# How to create a multi-node Kubernetes local cluster for CKA, CKAD, CKS exams!

For this demo, we will create 2 Ubuntu nodes using [Multipass](https://multipass.run/). Also, we can use this cluster environment to practice for CKAD/CKA/CKS exams, because uses Ubuntu nodes.

## 1. Installing Multipass

We can install it on Linux, Windows and MacOS.

| OS | Reference |
| ------ | ------ |
| Windows | https://multipass.run/docs/installing-on-windows |
| MacOS | https://multipass.run/docs/installing-on-macos |
| Linux | https://multipass.run/docs/installing-on-linux |

I will use Homebrew for MacOS:

```
brew install --cask multipass
```

Verify the installation:

```
multipass
```

![Alt text](images/multipass.png?raw=true "Multipass CLI")

## 2. Create our Linux nodes (Ubuntu)

> You should consider:

> - 2 GiB or more of RAM per machine--any less leaves little room for your apps.
> - At least 2 CPUs on the machine that you use as a control-plane node.

Launching master node:

```
multipass launch --name master --mem 2G --cpus 2 --disk 10G
```

Launching worker node:

```
multipass launch --name worker --mem 2G --cpus 2 --disk 10G
```

Verify nodes:

```
multipass ls
```

![Alt text](images/vms.png?raw=true "Ubuntu VMs")

## 3. Login to the master node and install requirements

> I installed k8s `v1.22` for simulating CKA/CKAD/CKS exams, you can use another version if needed. Use `crictl` to manage containers inside the nodes (Docker does not work)

```
multipass shell master
```

Scale to root user:

```
sudo -i
```

Run those commands:
```
apt-get upgrade
apt-get update
apt-get install -y apt-transport-https ca-certificates curl

curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" |  tee /etc/apt/sources.list.d/kubernetes.list

apt-get update
apt-get install -y kubeadm=1.22.4-00 kubelet=1.22.4-00 kubectl=1.22.4-00
apt-mark hold kubelet kubeadm kubectl

apt-get install containerd -y
mkdir -p /etc/containerd
containerd config default  /etc/containerd/config.toml
```

We need verify this file `/etc/sysctl.conf` and add:

`net.bridge.bridge-nf-call-iptables = 1`

After that, run:

```
sudo -s
sysctl --system
modprobe overlay
modprobe br_netfilter
hostnamectl set-hostname master
swapoff -a
echo '1' > /proc/sys/net/ipv4/ip_forward
exit
```

## 4. Initialize and configure the control plane node on master

```
kubeadm init
```

## 5. Setting up Kubeconfig file

```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

## 6. Install Weave Net for Cluster Networking

```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

## 7. Generate a token to join our worker node

```
kubeadm token generate
kubeadm token create [[TOKEN]] --print-join-command
```

*Copy and save the output to run on worker node*

![Alt text](images/join.png?raw=true "Ubuntu VMs")

## 8. Login to the worker node and install requirements

```
multipass shell worker
```

Scale to root user:

```
sudo -i
```

Run those commands:
```
apt-get upgrade
apt-get update
apt-get install -y apt-transport-https ca-certificates curl

curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" |  tee /etc/apt/sources.list.d/kubernetes.list

apt-get update
apt-get install -y kubeadm=1.22.4-00 kubelet=1.22.4-00
apt-mark hold kubelet kubeadm

apt-get install containerd -y
mkdir -p /etc/containerd
containerd config default  /etc/containerd/config.toml
```

We need verify this file `/etc/sysctl.conf` and add:

`net.bridge.bridge-nf-call-iptables = 1`

After that, run:

```
sudo -s
sysctl --system
modprobe overlay
modprobe br_netfilter
hostnamectl set-hostname worker
swapoff -a
echo '1' > /proc/sys/net/ipv4/ip_forward
exit
```
## 9. Join worker node to the cluster

Paste the output saved in **step 6**, something like this:

`kubeadm join 192.168.64.3:6443 --token dmxxxc.o6n5qh3d561c568n --discovery-token-ca-cert-hash sha256:5ad0a04d3c5f3a1cadcc0720975b40359f58y1c44442000bd312785h1ea72ffc`

## 10. Login to the master node and test your cluster

```
multipass shell master
sudo -i
kubectl get nodes -o wide
```

Kubernetes is ready. Now, you can practice on your multi-node cluster.

![Alt text](images/cluster.png?raw=true "Kubernetes cluster")


## 11. Delete cluster

If you need to delete your cluster, run these commands:

```
multipass stop master worker
multipass delete master worker
```

Happy hacking :smile: !!