# Goshen Technologies
# Kubernetes Cluster on Ubuntu with kubeadm

This guide sets up a 3-node Kubernetes cluster (**1 control plane + 2 workers**) on Ubuntu servers using **kubeadm**, **containerd** as the container runtime and **Flannel** for pod networking. It was tested on AWS EC2 Ubuntu instances with Kubernetes **v1.36**. The guide ends by deploying a sample app, **Zam Market**, exposed through a NodePort service.

## Prerequisites

| Item | Requirement |
|---|---|
| OS | Ubuntu 22.04 / 24.04 |
| Control plane | ≥ 2 vCPU, 2 GB RAM (e.g. `t3.medium`) |
| Workers | ≥ 2 vCPU, 2 GB RAM (e.g. `t3.small` or larger) |
| Network | All nodes can reach each other over their private IPs |

### AWS security group

Attach the same security group to all 3 nodes, with these inbound rules:

| Type | Port | Source | Purpose |
|---|---|---|---|
| All traffic | All | The security group itself | Node-to-node traffic (API, kubelet, etcd, Flannel VXLAN) |
| SSH | 22 | Your IP | Admin access |
| Custom TCP | 30080 | Your IP | Access to the Zam Market sample app |

---

## 1. Prepare all nodes

Run on **every node** (control plane and workers).

```bash
# Hostnames (run the matching one on each node)
sudo hostnamectl set-hostname k8s-master   # k8s-worker1 / k8s-worker2

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Kernel modules and sysctl
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay && sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# containerd
sudo apt-get update
sudo apt-get install -y containerd apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl enable containerd

# kubeadm, kubelet, kubectl
K8S_MINOR=v1.36
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

Verify:

```bash
kubeadm version && containerd --version
```

> **Tip:** If a long paste gets cut off, save the block above as `k8s-prep.sh` and run it with `sudo bash k8s-prep.sh`.

---

## 2. Initialise the control plane

Run on the **control plane only**, once.

```bash
MASTER_IP=$(hostname -I | awk '{print $1}')   # private IP
sudo kubeadm init \
  --apiserver-advertise-address=$MASTER_IP \
  --pod-network-cidr=10.244.0.0/16

# kubectl access for your user
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Pod network (Flannel's default CIDR matches the one above)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Copy the `kubeadm join ...` command that `kubeadm init` prints at the end.

---

## 3. Join the workers

Run on **each worker** with `sudo`, using the values from your own `kubeadm init` output:

```bash
sudo kubeadm join <MASTER_PRIVATE_IP>:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

Tokens expire after 24 hours. To generate a new join command on the control plane:

```bash
kubeadm token create --print-join-command
```

---

## 4. Verify the cluster

Run on the control plane.

```bash
# All nodes should be Ready
kubectl get nodes

# Optional: label the workers (only changes what the ROLES column shows)
kubectl label node k8s-worker1 node-role.kubernetes.io/worker=
kubectl label node k8s-worker2 node-role.kubernetes.io/worker=

# System pods: coredns and kube-flannel should all be Running
kubectl get pods -A
```

### Smoke test

```bash
kubectl create deploy nginx --image=nginx --replicas=2
kubectl expose deploy nginx --port=80 --type=NodePort

kubectl get pods -o wide   # both Running, each with a 10.244.x.x IP, spread across workers
kubectl get svc nginx      # note the NodePort (3xxxx)

# Should print the start of the nginx welcome page
NODE_IP=$(kubectl get node k8s-worker1 -o jsonpath='{.status.addresses[0].address}')
NODE_PORT=$(kubectl get svc nginx -o jsonpath='{.spec.ports[0].nodePort}')
curl -s http://$NODE_IP:$NODE_PORT | head -5
```

### DNS test

```bash
kubectl run dnstest --image=busybox:1.36 --rm -it --restart=Never \
  -- nslookup kubernetes.default.svc.cluster.local
# Expected: Address: 10.96.0.1
```

> busybox's `nslookup` can return `NXDOMAIN` for the short name `kubernetes.default`. Use the full name as shown above.

Clean up:

```bash
kubectl delete deploy nginx && kubectl delete svc nginx
```

---

## 5. Deploy the sample app: Zam Market

`zam-market.yaml` contains a Namespace, a ConfigMap (the web page), a Deployment (2 nginx replicas) and a NodePort Service on port **30080**.

Copy the manifest from your laptop to the control plane:

```bash
scp -i <your-key>.pem zam-market.yaml ubuntu@<MASTER_PUBLIC_IP>:~/
```

Deploy it:

```bash
kubectl apply -f zam-market.yaml
kubectl -n zam-market get pods -o wide
kubectl -n zam-market get svc
```

Open it in a browser:

```
http://<PUBLIC_IP_OF_ANY_NODE>:30080
```

The page footer shows the pod that served the request. Refresh a few times to watch traffic move between the replicas.

### Manage the app

```bash
kubectl -n zam-market scale deploy zam-market --replicas=4   # scale out
kubectl -n zam-market rollout restart deploy zam-market      # redeploy after editing the ConfigMap
kubectl delete -f zam-market.yaml                            # remove everything
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `kubeadm: command not found` | Step 1 didn't finish. Re-run it and check that `apt-get update` shows no errors for `pkgs.k8s.io`. |
| `user is not running as root` on join | Prefix `kubeadm join` with `sudo`. |
| Nodes stay `NotReady` | Check the Flannel pods (`kubectl -n kube-flannel get pods`) and make sure UDP 8472 is open between nodes. |
| Join times out | Use the control plane's **private** IP and make sure port 6443 is reachable. |
| Start a node over | `sudo kubeadm reset -f && sudo rm -rf /etc/cni/net.d $HOME/.kube` |
