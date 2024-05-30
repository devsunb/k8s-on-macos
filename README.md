# Kubernetes on macOS (Apple Silicon)

Describe how to configure a Kubernetes environment for local testing on macOS Apple Silicon.

# Background

As a DevOps Engineer working with MacBook, I need to configure a local Kubernetes Cluster for testing.

But VM is required to run docker engine in macOS, and there is too many tools for creating vm and configuring docker/kubernetes environment.

I tried multipass, utm, lima, colima, minikube, kind, k3d and decided to use below

- a fully functional kubernetes cluster (kube-vip, 3 CPs, 3 Workers) based on lima and kubeadm for learning kubernetes architecture.
- colima docker + minikube kubernetes cluster (1 CP, can add worker node) for daily work, testing multi-node (Affinity, PodDisruptionBudge, cordon, drain, ...), and testing multiple kubernetes versions

# lima

Source: <https://github.com/louhisuo/k8s-on-macos>

## Versions

- Lima VM 0.22.0 / socket_vmnet 1.1.4 - Virtualization
- Debian 12 - Node images
- Kubernetes 1.30.1 - Kubernetes release
- Cilium 1.15.4 - CNI, L2 LB, L7 LB (Ingress Controller) and L4/L7 LB (Gateway API)
- Gateway API 1.1 - CRDs supported by Cilium 1.15.4
- kube-vip 0.8.0 - Kubernetes Control Plane LB
- metrics-server 0.7.1
- local-path-provisioner 0.0.26

## Prerequisites

### Required Tools

Following tools are required by this project.

- [lima](https://github.com/lima-vm/lima)
- [socket_vmnet](https://github.com/lima-vm/socket_vmnet/)
- [kubernetes-cli](https://github.com/kubernetes/kubectl)
- [helm](https://helm.sh/)
- [git](https://git-scm.com/)
- [cilium-cli](https://github.com/cilium/cilium-cli/)

[Homebrew](https://brew.sh/) can be used to install these tools on macOS host.

```sh
brew install lima socket_vmnet kubernetes-cli helm git
```

`cilium-cli` macOS install script can be found on [Cilium Getting Started Doc](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/)

```sh
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "arm64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-darwin-${CLI_ARCH}.tar.gz{,.sha256sum}
shasum -a 256 -c cilium-darwin-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-darwin-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-darwin-${CLI_ARCH}.tar.gz{,.sha256sum}
```

### Networking

Shared network `192.168.105.0/24` in macOS is used as it allows both Host-VM and VM-VM communication.
By default Lima VM uses DHCP range until `192.168.105.254` therefore we use IP address range
from `192.168.105.100` and onward in our Kubernetes setup.
To have predictable node IPs for a Kubernetes cluster,
it is neccessary to [reserve IPs](https://github.com/lima-vm/socket_vmnet#how-to-reserve-dhcp-addresses) to be used from DHCP server in macOS.

#### Kubernetes node IP range

Define following [/etc/bootptab](./etc/bootptab) file.

```sh
# WARNING: this will overwrite your bootptab
sudo cp etc/bootptab /etc/bootptab
```

| Host     | MAC Address       | IP address      |
| -------- | ----------------- | --------------- |
| cp       | 00:00:00:00:00:00 | 192.168.105.100 |
| cp-1     | 00:00:00:00:00:01 | 192.168.105.101 |
| cp-2     | 00:00:00:00:00:02 | 192.168.105.102 |
| cp-3     | 00:00:00:00:00:03 | 192.168.105.103 |
| worker-1 | 10:00:00:00:00:01 | 192.168.105.201 |
| worker-2 | 10:00:00:00:00:02 | 192.168.105.202 |
| worker-3 | 10:00:00:00:00:03 | 192.168.105.203 |

If DHCP daemon already running, reload it.

Before first VM start, `system/com.apple.bootpd` service might not exist.

```sh
sudo /bin/launchctl print system/com.apple.bootpd # if state = running, run below
sudo /bin/launchctl kickstart -kp system/com.apple.bootpd
```

#### Kubernetes API server

Kubernetes API server is available via VIP address `192.168.105.100`.

#### L2 Aware load balancer, Ingress and Gateway

IP address pool for `Load Balancer` services must be configured to same shared subnet than cluster `node IPs`. Currently L2 Aware LB in Cilium CNI is used and default address pool is configured is `192.168.105.240/28`.

From the assigned address pool following IPs are "reserved", leaving 12 addresses available for different LB services.

- `192.168.105.241` is assigned to `Ingress` (Cilium Ingress Controller) and
- `192.168.105.242` is reserved for `Gateway` (Cilium Gateway API). `Gateway` configuration is work in progress.

## Misc

### Exposing services via NodePort to macOS host

It is possible to expose Kubernetes services via `NodePort` to `macOS` host. Full `NodePort` range `30000-32767` is exposed to `macOS` host from provisioned `Lima VM` machines during machine creation phaase.

Actual services with `type: NodePort` will be available on `macOS` host via `node IP` address of any Control Plane or Worker nodes of a cluster (not via VIP address) and assigned `NodePort` value for a service.

## Setup

### Set lima sudoers

```sh
limactl sudoers | sudo tee /private/etc/sudoers.d/lima
```

### Create VMs

```sh
# if you create lima vm for the first time, lima will download vm image, so wait for the first vm to create before creating the others or you will get a file error.
limactl create --set='.networks[].macAddress="00:00:00:00:00:01"' --name cp-1 lima.yaml --tty=false
limactl create --set='.networks[].macAddress="00:00:00:00:00:02"' --name cp-2 lima.yaml --tty=false
limactl create --set='.networks[].macAddress="00:00:00:00:00:03"' --name cp-3 lima.yaml --tty=false
limactl create --set='.networks[].macAddress="10:00:00:00:00:01"' --name worker-1 lima.yaml --tty=false
limactl create --set='.networks[].macAddress="10:00:00:00:00:02"' --name worker-2 lima.yaml --tty=false
limactl create --set='.networks[].macAddress="10:00:00:00:00:03"' --name worker-3 lima.yaml --tty=false

# if you start lima vm for the first time, lima will create user config file, so wait for the first vm to run before starting the others or you will get a file error.
# and if you're staring more than one vm, start them at the same time, or start the other vms after the startup is complete.
# Starting a vm when another vm is in the middle of starting causes a network error.
limactl start cp-1
limactl start cp-2
limactl start cp-3
limactl start worker-1
limactl start worker-2
limactl start worker-3

limactl shell cp-1
limactl shell cp-2
limactl shell cp-3
limactl shell worker-1
limactl shell worker-2
limactl shell worker-3
```

Run in all vms

```sh
sudo apt update && sudo apt upgrade -y

# Network configuration
cat <<EOF | sudo tee -a /etc/hosts
192.168.105.101 cp-1
192.168.105.102 cp-2
192.168.105.103 cp-3
192.168.105.201 worker-1
192.168.105.202 worker-2
192.168.105.203 worker-3
EOF

cat <<EOF | sudo tee -a /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee -a /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# Install containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's#SystemdCgroup = false#SystemdCgroup = true#' /etc/containerd/config.toml
sudo sed -i 's#sandbox_image = "registry.k8s.io/pause:3.8"#sandbox_image = "registry.k8s.io/pause:3.9"#' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Install kubernetes
K8S_REPO=v1.30
K8S_VERSION=1.30.1

sudo apt install -y apt-transport-https ca-certificates curl gpg ipvsadm jq

curl -fsSL https://pkgs.k8s.io/core:/stable:/${K8S_REPO}/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/'${K8S_REPO}'/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubeadm=${K8S_VERSION}-* kubectl=${K8S_VERSION}-* kubelet=${K8S_VERSION}-*
sudo apt-mark hold kubeadm kubectl kubelet
sudo systemctl enable --now kubelet
```

### Initiate Kubernetes Cluster

#### Initiate Kubernetes Control Plane

run in each cp vms

```sh
export KVVERSION=v0.8.0
export INTERFACE=lima0
export VIP=192.168.105.100
sudo ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION
sudo ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip manifest pod \
    --arp \
    --controlplane \
    --address $VIP \
    --interface $INTERFACE \
    --enableLoadBalancer \
    --leaderElection | sudo tee /etc/kubernetes/manifests/kube-vip.yaml

# Kubernetes >=1.29 kube-vip workaround https://github.com/kube-vip/kube-vip/issues/684#issuecomment-1864855405
sudo sed -i 's#path: /etc/kubernetes/admin.conf#path: /etc/kubernetes/super-admin.conf#' /etc/kubernetes/manifests/kube-vip.yaml

# run next line only in cp-1 and it should be done before running cp-2, cp-3 join command
sudo kubeadm init --upload-certs --config manifests/kubeadm/cp-1-init-cfg.yaml
# run next line only in cp-2
sudo kubeadm join --config manifests/kubeadm/cp-2-join-cfg.yaml
# run next line only in cp-3
sudo kubeadm join --config manifests/kubeadm/cp-3-join-cfg.yaml

sudo sed -i 's#path: /etc/kubernetes/super-admin.conf#path: /etc/kubernetes/admin.conf#' /etc/kubernetes/manifests/kube-vip.yaml

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```sh
# combine or replace ~/.kube/config in macOS
limactl cp cp-1:.kube/config kubeconfig
```

#### Join worker nodes to Kubernetes cluster

run in each worker vms

```sh
# run next line only in worker-1
sudo kubeadm join --config manifests/kubeadm/worker-1-join-cfg.yaml
# run next line only in worker-2
sudo kubeadm join --config manifests/kubeadm/worker-2-join-cfg.yaml
# run next line only in worker-3
sudo kubeadm join --config manifests/kubeadm/worker-3-join-cfg.yaml
```

### Post Cluster Creation Steps

#### Manual approval of kubelet serving certificates

Approve any pending `kubelet-serving` certificate

```sh
kubectl get csr
kubectl get csr | grep "Pending" | awk '{print $1}' | xargs kubectl certificate approve
```

#### Install Cilium CNI

Install Gateway API CRDs supported by Cilium.

<https://docs.cilium.io/en/latest/network/servicemesh/gateway-api/gateway-api/>

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.1.0/config/crd/standard/gateway.networking.k8s.io_gatewayclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.1.0/config/crd/standard/gateway.networking.k8s.io_gateways.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.1.0/config/crd/standard/gateway.networking.k8s.io_httproutes.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.1.0/config/crd/standard/gateway.networking.k8s.io_referencegrants.yaml
# there is config/crd/standard/gateway.networking.k8s.io_grpcroutes.yaml in v1.1.0, but if use it an error occurs when cilium init
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.1.0/config/crd/experimental/gateway.networking.k8s.io_grpcroutes.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.1.0/config/crd/experimental/gateway.networking.k8s.io_tlsroutes.yaml
```

Install CNI (Cilium) with L2 LB, L7 LB (Ingress Controller) and L4/L7 LB (Gateway API) support enabled.

```sh
helm repo add cilium https://helm.cilium.io/
helm repo update
helm install cilium cilium/cilium --version 1.15.5 --namespace kube-system --values manifests/cilium/values.yaml
```

Configure L2 announcements and address pool for L2 aware Load Balancer

```sh
kubectl apply -f manifests/cilium/l2-lb-cfg.yaml
```

Configure Gateway (default)

```sh
kubectl apply -f manifests/cilium/gtw-cfg.yaml
```

#### Install Metrics server add-on

Install metrics server

```sh
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

#### Install Local path provisioner CSI

Install local path provisioner

```sh
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml
```

Apply local path provisioner configuration. There will be a delay after `configmap` is applied before the provisioner picks it up.

```sh
kubectl apply -f manifests/local-path-provisioner/provisioner-cm.yaml
```

### Finals checks

Check from that cluster works as expected

```sh
kubectl version
cilium status
kubectl get --raw='/readyz?verbose'
```

# UNAVAILABLE: tart

I tried to use tart instead of lima, but tart is **unavailable because it does not seem to support inter-VM connections**

Keep just for the record

## Versions

- Tart Virtualization 2.11.0
- Debian 12
- Kubernetes 1.29.4
- Cilium 1.15.4
- Gateway API 1.1
- kube-vip 0.8.0
- metrics-server 0.7.1
- local-path-provisioner 0.0.26

## Prerequisites

### Required Tools

Following tools are required by this project.

- [tart](https://github.com/cirruslabs/tart/)
  - [sshpass](https://tart.run/quick-start/#ssh-access)
- [kubernetes-cli](https://github.com/kubernetes/kubectl)
- [helm](https://helm.sh/)
- [git](https://git-scm.com/)
- [cilium-cli](https://github.com/cilium/cilium-cli/)

[Homebrew](https://brew.sh/) can be used to install these tools on macOS host.

```sh
brew install tart cirruslabs/cli/sshpass kubernetes-cli helm git
```

`cilium-cli` macOS install script can be found on [Cilium Getting Started Doc](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/)

```sh
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "arm64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-darwin-${CLI_ARCH}.tar.gz{,.sha256sum}
shasum -a 256 -c cilium-darwin-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-darwin-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-darwin-${CLI_ARCH}.tar.gz{,.sha256sum}
```

### Networking

#### Kubernetes node IP range

| Host     | IP address      |
| -------- | --------------- |
| cp       | 192.168.106.100 |
| cp-1     | 192.168.106.101 |
| cp-2     | 192.168.106.102 |
| cp-3     | 192.168.106.103 |
| worker-1 | 192.168.106.104 |
| worker-2 | 192.168.106.105 |
| worker-3 | 192.168.106.106 |

#### Kubernetes API server

Kubernetes API server is available via VIP address `192.168.106.100`.

#### L2 Aware load balancer, Ingress and Gateway

IP address pool for `Load Balancer` services must be configured to same shared subnet than cluster `node IPs`. Currently L2 Aware LB in Cilium CNI is used and default address pool is configured is `192.168.106.240/28`.

From the assigned address pool following IPs are "reserved", leaving 12 addresses available for different LB services.

- `192.168.106.241` is assigned to `Ingress` (Cilium Ingress Controller) and
- `192.168.106.242` is reserved for `Gateway` (Cilium Gateway API). `Gateway` configuration is work in progress.

## Setup

### Set ssh config

`~/.ssh/config`

`PreferredAuthentications` is required when you use 1password ssh agent

```
Host 192.168.106.* cp-* worker-*
  User admin
  PreferredAuthentications password
```

### Set /etc/hosts

`/etc/hosts`

```
192.168.106.101 cp-1
192.168.106.102 cp-2
192.168.106.103 cp-3
192.168.106.104 worker-1
192.168.106.105 worker-2
192.168.106.106 worker-3
```

### Create VMs

```sh
tart clone ghcr.io/cirruslabs/debian:latest cp-1
tart clone ghcr.io/cirruslabs/debian:latest cp-2
tart clone ghcr.io/cirruslabs/debian:latest cp-3
tart clone ghcr.io/cirruslabs/debian:latest worker-1
tart clone ghcr.io/cirruslabs/debian:latest worker-2
tart clone ghcr.io/cirruslabs/debian:latest worker-3

# default: cpu 4, memory 4096, disk-size 20
tart set worker-1 --disk-size 100
tart set worker-2 --disk-size 100
tart set worker-3 --disk-size 100

cat <<EOF | tee ~/.tart/vms/cp-1/config.json
{
  "version": 1,
  "os": "linux",
  "arch": "arm64",
  "cpuCount": 2,
  "cpuCountMin": 4,
  "memorySize": 2147483648,
  "memorySizeMin": 4294967296,
  "display": {
    "width": 1024,
    "height": 768
  },
  "macAddress": "10:00:00:00:00:01"
}
EOF
cat <<EOF | tee ~/.tart/vms/cp-2/config.json
{
  "version": 1,
  "os": "linux",
  "arch": "arm64",
  "cpuCount": 2,
  "cpuCountMin": 4,
  "memorySize": 2147483648,
  "memorySizeMin": 4294967296,
  "display": {
    "width": 1024,
    "height": 768
  },
  "macAddress": "10:00:00:00:00:02"
}
EOF
cat <<EOF | tee ~/.tart/vms/cp-3/config.json
{
  "version": 1,
  "os": "linux",
  "arch": "arm64",
  "cpuCount": 2,
  "cpuCountMin": 4,
  "memorySize": 2147483648,
  "memorySizeMin": 4294967296,
  "display": {
    "width": 1024,
    "height": 768
  },
  "macAddress": "10:00:00:00:00:03"
}
EOF
cat <<EOF | tee ~/.tart/vms/worker-1/config.json
{
  "version": 1,
  "os": "linux",
  "arch": "arm64",
  "cpuCount": 4,
  "cpuCountMin": 4,
  "memorySize": 4294967296,
  "memorySizeMin": 4294967296,
  "display": {
    "width": 1024,
    "height": 768
  },
  "macAddress": "10:00:00:00:00:04"
}
EOF
cat <<EOF | tee ~/.tart/vms/worker-2/config.json
{
  "version": 1,
  "os": "linux",
  "arch": "arm64",
  "cpuCount": 4,
  "cpuCountMin": 4,
  "memorySize": 4294967296,
  "memorySizeMin": 4294967296,
  "display": {
    "width": 1024,
    "height": 768
  },
  "macAddress": "10:00:00:00:00:05"
}
EOF
cat <<EOF | tee ~/.tart/vms/worker-3/config.json
{
  "version": 1,
  "os": "linux",
  "arch": "arm64",
  "cpuCount": 4,
  "cpuCountMin": 4,
  "memorySize": 4294967296,
  "memorySizeMin": 4294967296,
  "display": {
    "width": 1024,
    "height": 768
  },
  "macAddress": "10:00:00:00:00:06"
}
EOF

nohup tart run --no-graphics cp-1 &>/dev/null &
nohup tart run --no-graphics cp-2 &>/dev/null &
nohup tart run --no-graphics cp-3 &>/dev/null &
nohup tart run --no-graphics worker-1 &>/dev/null &
nohup tart run --no-graphics worker-2 &>/dev/null &
nohup tart run --no-graphics worker-3 &>/dev/null &

# ssh with initial ip
sshpass -p admin ssh $(tart ip cp-1)
sshpass -p admin ssh $(tart ip cp-2)
sshpass -p admin ssh $(tart ip cp-3)
sshpass -p admin ssh $(tart ip worker-1)
sshpass -p admin ssh $(tart ip worker-2)
sshpass -p admin ssh $(tart ip worker-3)
```

run in each vm

```sh
sudo apt update && sudo apt upgrade -y && sudo apt install -y locales-all

# install kubeadm, kubelet, kubectl
# https://v1-29.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
sudo apt install -y containerd apt-transport-https gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet

echo "network: {config: disabled}" | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
sudo rm /etc/netplan/50-cloud-init.yaml

# use network.ethernets.enp0s1.addresses[0] to match ip table above
cat <<EOF | sudo tee /etc/netplan/60-static-ip.yaml
network:
  version: 2
  ethernets:
    enp0s1:
      dhcp4: false
      addresses:
        - 192.168.106.101/24
      routes:
        - to: default
          via: 192.168.106.1
      nameservers:
        addresses:
          - 192.168.106.1
EOF
cat <<EOF | sudo tee /etc/netplan/60-static-ip.yaml
network:
  version: 2
  ethernets:
    enp0s1:
      dhcp4: false
      addresses:
        - 192.168.106.102/24
      routes:
        - to: default
          via: 192.168.106.1
      nameservers:
        addresses:
          - 192.168.106.1
EOF
cat <<EOF | sudo tee /etc/netplan/60-static-ip.yaml
network:
  version: 2
  ethernets:
    enp0s1:
      dhcp4: false
      addresses:
        - 192.168.106.103/24
      routes:
        - to: default
          via: 192.168.106.1
      nameservers:
        addresses:
          - 192.168.106.1
EOF
cat <<EOF | sudo tee /etc/netplan/60-static-ip.yaml
network:
  version: 2
  ethernets:
    enp0s1:
      dhcp4: false
      addresses:
        - 192.168.106.104/24
      routes:
        - to: default
          via: 192.168.106.1
      nameservers:
        addresses:
          - 192.168.106.1
EOF
cat <<EOF | sudo tee /etc/netplan/60-static-ip.yaml
network:
  version: 2
  ethernets:
    enp0s1:
      dhcp4: false
      addresses:
        - 192.168.106.105/24
      routes:
        - to: default
          via: 192.168.106.1
      nameservers:
        addresses:
          - 192.168.106.1
EOF
cat <<EOF | sudo tee /etc/netplan/60-static-ip.yaml
network:
  version: 2
  ethernets:
    enp0s1:
      dhcp4: false
      addresses:
        - 192.168.106.106/24
      routes:
        - to: default
          via: 192.168.106.1
      nameservers:
        addresses:
          - 192.168.106.1
EOF

sudo chmod 600 /etc/netplan/60-static-ip.yaml
sudo sync # if you stop vm without run this, file changes may not applied

cat <<EOF | sudo tee -a /etc/hosts
192.168.106.101 cp-1
192.168.106.102 cp-2
192.168.106.103 cp-3
192.168.106.104 worker-1
192.168.106.105 worker-2
192.168.106.106 worker-3
EOF

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's#SystemdCgroup = false#SystemdCgroup = true#' /etc/containerd/config.toml
sudo sed -i 's#sandbox_image = "registry.k8s.io/pause:3.8"#sandbox_image = "registry.k8s.io/pause:3.9"#' /etc/containerd/config.toml
sudo systemctl restart containerd
```

```sh
tart stop cp-1
tart stop cp-2
tart stop cp-3
tart stop worker-1
tart stop worker-2
tart stop worker-3

# start vm with dir mount
nohup tart run --no-graphics --dir="~/path/to/k8s-on-macos" cp-1 &>/dev/null &
nohup tart run --no-graphics --dir="~/path/to/k8s-on-macos" cp-2 &>/dev/null &
nohup tart run --no-graphics --dir="~/path/to/k8s-on-macos" cp-3 &>/dev/null &
nohup tart run --no-graphics --dir="~/path/to/k8s-on-macos" worker-1 &>/dev/null &
nohup tart run --no-graphics --dir="~/path/to/k8s-on-macos" worker-2 &>/dev/null &
nohup tart run --no-graphics --dir="~/path/to/k8s-on-macos" worker-3 &>/dev/null &

# now you can ssh with static ip
sshpass -p admin ssh cp-1
sshpass -p admin ssh cp-2
sshpass -p admin ssh cp-3
sshpass -p admin ssh worker-1
sshpass -p admin ssh worker-2
sshpass -p admin ssh worker-3

# --- run in each vm
ip addr # check ip
# mount host ~/dev/k8s to init k8s cluster
sudo mkdir /mnt/shared
sudo mount -t virtiofs com.apple.virtio-fs.automount /mnt/shared # if you reboot vm, you shoud re-run this
# --- run in each vm
```

### Initiate Kubernetes Cluster

#### Initiate Kubernetes Control Planes

```sh
# run in cp
export KVVERSION=v0.8.0
export INTERFACE=enp0s1
export VIP=192.168.106.100
sudo mkdir -p /etc/kubernetes/manifests
sudo ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION
sudo ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip manifest pod \
    --arp \
    --controlplane \
    --address $VIP \
    --interface $INTERFACE \
    --enableLoadBalancer \
    --leaderElection | sudo tee /etc/kubernetes/manifests/kube-vip.yaml

# Kubernetes >=1.29 kube-vip workaround https://github.com/kube-vip/kube-vip/issues/684#issuecomment-1864855405
sudo sed -i 's#path: /etc/kubernetes/admin.conf#path: /etc/kubernetes/super-admin.conf#' /etc/kubernetes/manifests/kube-vip.yaml

# --- NOT WORKING ---
# --- run in cp-1 only
sudo kubeadm init --upload-certs --config /mnt/shared/manifests/kubeadm/cp-1-init-cfg.yaml
# --- run in cp-1 only
# --- run in cp-2 only
sudo kubeadm join --config /mnt/shared/manifests/kubeadm/cp-2-join-cfg.yaml
# --- run in cp-2 only
# --- run in cp-3 only
sudo kubeadm join --config /mnt/shared/manifests/kubeadm/cp-3-join-cfg.yaml
# --- run in cp-3 only
```

# minikube

Single VM to Docker Runtime (colima), minikube with docker

Pros over colima

- Multiple worker node
- Various kubernetes versions

To use minikube docker driver, docker should be configured.

```sh
colima start -c 4 -m 4
# -c 4: VM with 4 CPUs (default 2)
# -m 4: VM with 4 GiB Memory (default 2)
```

docker context will be automatically configured

```sh
mk start -n 2 --kubernetes-version 1.30
# -n 2: 2 nodes (Single CP/Worker and n-1 Workers)
```

`~/.kube/config` will be automatically modified

# colima

I tried to use colima single node kubernetes cluster base on k3s for daily work but with basic setup, kubectl to colima kubernetes cluster while lima vms running occurs tls error:

```
Unable to connect to the server: tls: failed to verify certificate: x509: certificate signed by unknown authority
```

So now I use minikube instead.

Pros over minikube

- create/start/stop/delete is simpler and faster
- LoadBalancer Service is accessible with localhost without `minikube tunnel`

## Setup

```sh
colima start -k -c 4 -m 4
# -k: start single node (k3s in docker) kubernetes cluster
# -c 4: VM with 4 CPUs (default 2)
# -m 4: VM with 4 GiB Memory (default 2)
```

docker context and `~/.kube/config` will be automatically modified
