
# Project Title
Kubernetes Cluster Setup on Local Computer using Vagrant, Git Bash, and kubeadm with Application Deployment
## Description:

This project demonstrates the creation of a Kubernetes cluster on a local machine using **Vagrant** and **Git Bash**. The cluster is configured with **kubeadm, kubectl, kubelet,** and the **CRI-O** container runtime, along with **Calico** as the CNI (Container Network Interface) for network management. Two applications are deployed: **NGINX** and a **React.js mobile shop application.** Both applications are exposed using **NodePort,** allowing access from the local machine.

## Tools and Technologies Used
- Vagrant
- Git Bash
- Docker
- DockerHub
- kubeadm
- kubectl
- kubelet
- CRI-O
- Calico
- NGINX
- React.js
- NodePort

## Flow Diagram

![P07](https://github.com/user-attachments/assets/078d2487-fc89-4353-8073-b07b712a26a8)

# Components and Tools

## Vagrant
Vagrant is used to set up virtual machines (VMs) on the local computer, providing the infrastructure needed to create a Kubernetes cluster.

## Git Bash
The command-line interface used to manage the local environment, run Vagrant commands, and interact with the Kubernetes cluster using kubectl.

## kubeadm
A Kubernetes tool used to initialize and configure the Kubernetes cluster. It handles the setup of the master node and joins worker nodes.

## kubectl
Command-line tool used to manage and interact with the Kubernetes cluster, deploy applications, and check the status of pods and services.

## kubelet
Responsible for ensuring containers are running as expected in the Kubernetes cluster.

## CRI-O Runtime
A lightweight, secure container runtime used as an alternative to Docker for running containers in Kubernetes.

## Calico
A networking solution that provides a CNI for Kubernetes, ensuring proper network communication between pods and services within the cluster.

## Kubeadm Configuration and Images
kubeadm is used to configure the Kubernetes cluster and manage images required for the control plane components.

The Kubernetes master node is initialized with kubeadm, and worker nodes join the cluster to form a multi-node setup.

## Application Deployment
**NGINX:** A web server application is deployed as a Kubernetes pod, serving static content.

**React.js** Mobile Shop Application: A full-fledged e-commerce application is deployed in another pod, built with React.js for the frontend, simulating a mobile shop interface.

## NodePort Access
Both the NGINX and React.js mobile shop applications are exposed using NodePort, allowing users to access the applications via specific ports on the local machine.

# Project Workflow
## Infrastructure Setup with Vagrant
Vagrant is used to create the virtual machines for the Kubernetes cluster.

VMs are provisioned with necessary resources and network configurations to allow communication between the nodes.

## Kubernetes Cluster Initialization with kubeadm
kubeadm is used to initialize the cluster on the master node.

Worker nodes are added to the cluster using kubeadm join commands.

CRI-O is set as the container runtime for running pods within the cluster.

## Network Configuration with Calico
Calico is deployed as the CNI plugin, enabling pod-to-pod communication and managing network policies within the Kubernetes cluster.

## Application Deployment
NGINX and the React.js mobile shop applications are containerized and deployed as Kubernetes pods.

kubectl is used to create deployments and expose the applications using NodePort.

## Accessing Applications via NodePort
Once deployed, the NGINX and React.js applications can be accessed through specific ports on the local machine by forwarding traffic via NodePort.

This allows users to test and interact with the applications directly on their local machine.

## Vagrant VM Creation
![Screenshot (663)](https://github.com/user-attachments/assets/ab52d98b-a12c-4722-b37d-775752b37ab9)
![Screenshot (662)](https://github.com/user-attachments/assets/125789ca-34ea-4af0-b450-37a540c8592e)

## SSH Key for Vagrant VM
![Screenshot (666)](https://github.com/user-attachments/assets/21c8b2eb-4996-4f5e-8105-da5c0b2a5ec4)
![Screenshot (667)](https://github.com/user-attachments/assets/b21d0b14-2da1-495c-bedf-34a6638a6e68)

## vagrantfile

Create Vagrant VMs

```bash
  require "yaml"
settings = YAML.load_file "settings.yaml"

IP_SECTIONS = settings["network"]["control_ip"].match(/^([0-9.]+\.)([^.]+)$/)
# First 3 octets including the trailing dot:
IP_NW = IP_SECTIONS.captures[0]
# Last octet excluding all dots:
IP_START = Integer(IP_SECTIONS.captures[1])
NUM_WORKER_NODES = settings["nodes"]["workers"]["count"]

Vagrant.configure("2") do |config|
  config.vm.provision "shell", env: { "IP_NW" => IP_NW, "IP_START" => IP_START, "NUM_WORKER_NODES" => NUM_WORKER_NODES }, inline: <<-SHELL
      apt-get update -y
      echo "$IP_NW$((IP_START)) master-node" >> /etc/hosts
      for i in `seq 1 ${NUM_WORKER_NODES}`; do
        echo "$IP_NW$((IP_START+i)) worker-node0${i}" >> /etc/hosts
      done
  SHELL

  if `uname -m`.strip == "aarch64"
    config.vm.box = settings["software"]["box"] + "-arm64"
  else
    config.vm.box = settings["software"]["box"]
  end
  config.vm.box_check_update = true

  config.vm.define "master" do |master|
    master.vm.hostname = "master-node"
    master.vm.network "private_network", ip: settings["network"]["control_ip"]
    if settings["shared_folders"]
      settings["shared_folders"].each do |shared_folder|
        master.vm.synced_folder shared_folder["host_path"], shared_folder["vm_path"]
      end
    end
    master.vm.provider "virtualbox" do |vb|
        vb.cpus = settings["nodes"]["control"]["cpu"]
        vb.memory = settings["nodes"]["control"]["memory"]
        if settings["cluster_name"] and settings["cluster_name"] != ""
          vb.customize ["modifyvm", :id, "--groups", ("/" + settings["cluster_name"])]
        end
    end
    master.vm.provision "shell",
      env: {
        "DNS_SERVERS" => settings["network"]["dns_servers"].join(" "),
        "ENVIRONMENT" => settings["environment"],
        "KUBERNETES_VERSION" => settings["software"]["kubernetes"],
        "OS" => settings["software"]["os"]
      },
      path: "E:/scripts/common.sh"
    master.vm.provision "shell",
      env: {
        "CALICO_VERSION" => settings["software"]["calico"],
        "CONTROL_IP" => settings["network"]["control_ip"],
        "POD_CIDR" => settings["network"]["pod_cidr"],
        "SERVICE_CIDR" => settings["network"]["service_cidr"]
      },
      path: "E:/scripts/master.sh"
  end

  (1..NUM_WORKER_NODES).each do |i|

    config.vm.define "node0#{i}" do |node|
      node.vm.hostname = "worker-node0#{i}"
      node.vm.network "private_network", ip: IP_NW + "#{IP_START + i}"
      if settings["shared_folders"]
        settings["shared_folders"].each do |shared_folder|
          node.vm.synced_folder shared_folder["host_path"], shared_folder["vm_path"]
        end
      end
      node.vm.provider "virtualbox" do |vb|
          vb.cpus = settings["nodes"]["workers"]["cpu"]
          vb.memory = settings["nodes"]["workers"]["memory"]
          if settings["cluster_name"] and settings["cluster_name"] != ""
            vb.customize ["modifyvm", :id, "--groups", ("/" + settings["cluster_name"])]
          end
      end
      node.vm.provision "shell",
        env: {
          "DNS_SERVERS" => settings["network"]["dns_servers"].join(" "),
          "ENVIRONMENT" => settings["environment"],
          "KUBERNETES_VERSION" => settings["software"]["kubernetes"],
          "OS" => settings["software"]["os"]
        },
        path: "E:/scripts/common.sh"
        node.vm.provision "shell", path: "E:/scripts/node.sh"
    end

  end
end 
```

## User Provisioning Yaml File

IP CPU Memory Box Type 

```bash
---
# cluster_name is used to group the nodes in a folder within VirtualBox:
cluster_name: Kubernetes Cluster
network:
  # Worker IPs are simply incremented from the control IP.
  control_ip: 10.0.0.10
  dns_servers:
    - 8.8.8.8
    - 1.1.1.1
  pod_cidr: 172.16.1.0/16
  service_cidr: 172.17.1.0/18
nodes:
  control:
    cpu: 2
    memory: 4096
  workers:
    count: 2
    cpu: 2
    memory: 4096

software:
  box: bento/ubuntu-22.04
  calico: 3.25.0
  kubernetes: 1.26.1-00
  os: xUbuntu_22.04

```
## Common Settings and Installation
Tools

```bash
#!/bin/bash
#
# Common setup for all servers (Control Plane and Nodes)

set -euxo pipefail

# Variable Declaration

# DNS Setting
if [ ! -d /etc/systemd/resolved.conf.d ]; then
 sudo mkdir /etc/systemd/resolved.conf.d/
fi
cat <<EOF | sudo tee /etc/systemd/resolved.conf.d/dns_servers.conf
[Resolve]
DNS=${DNS_SERVERS}
EOF

sudo systemctl restart systemd-resolved

# disable swap
sudo swapoff -a

# keeps the swap off during reboot
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
sudo apt-get update -y
# Install CRI-O Runtime

VERSION="$(echo ${KUBERNETES_VERSION} | grep -oE '[0-9]+\.[0-9]+')"

# Create the .conf file to load the modules at bootup
cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Set up required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system

cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /
EOF
cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list
deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /
EOF

curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -

sudo apt-get update
sudo apt-get install cri-o cri-o-runc -y

cat >> /etc/default/crio << EOF
${ENVIRONMENT}
EOF
sudo systemctl daemon-reload
sudo systemctl enable crio --now

echo "CRI runtime installed susccessfully"

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
sudo mkdir -p -m 755 /etc/apt/keyrings
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update -y
sudo apt-get install -y kubeadm=1.28.1-1.1 kubelet=1.28.1-1.1 kubectl=1.28.1-1.1
sudo apt-get update -y
sudo apt-get install -y jq

local_ip="$(ip --json a s | jq -r '.[] | if .ifname == "eth1" then .addr_info[] | if .family == "inet" then .local else empty end else empty end')"
cat > /etc/default/kubelet << EOF
KUBELET_EXTRA_ARGS=--node-ip=$local_ip
${ENVIRONMENT}
EOF
```

## Control Plane Configuration and Installation

```bash
#!/bin/bash
#
# Setup for Control Plane (Master) servers

set -euxo pipefail

NODENAME=$(hostname -s)

sudo kubeadm config images pull

echo "Preflight Check Passed: Downloaded All Required Images"

sudo kubeadm init --apiserver-advertise-address=$CONTROL_IP --apiserver-cert-extra-sans=$CONTROL_IP --pod-network-cidr=$POD_CIDR --service-cidr=$SERVICE_CIDR --node-name "$NODENAME" --ignore-preflight-errors Swap

mkdir -p "$HOME"/.kube
sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config

# Save Configs to shared /Vagrant location

# For Vagrant re-runs, check if there is existing configs in the location and delete it for saving new configuration.

config_path="/vagrant/configs"

if [ -d $config_path ]; then
  rm -f $config_path/*
else
  mkdir -p $config_path
fi

cp -i /etc/kubernetes/admin.conf $config_path/config
touch $config_path/join.sh
chmod +x $config_path/join.sh

kubeadm token create --print-join-command > $config_path/join.sh

# Install Calico Network Plugin

curl https://raw.githubusercontent.com/projectcalico/calico/v${CALICO_VERSION}/manifests/calico.yaml -O

kubectl apply -f calico.yaml

sudo -i -u vagrant bash << EOF
whoami
mkdir -p /home/vagrant/.kube
sudo cp -i $config_path/config /home/vagrant/.kube/
sudo chown 1000:1000 /home/vagrant/.kube/config
EOF
```

## worker node configuration

```bash
#!/bin/bash
#
# Setup for Node servers

set -euxo pipefail

config_path="/vagrant/configs"

/bin/bash $config_path/join.sh -v

sudo -i -u vagrant bash << EOF
whoami
mkdir -p /home/vagrant/.kube
sudo cp -i $config_path/config /home/vagrant/.kube/
sudo chown 1000:1000 /home/vagrant/.kube/config
NODENAME=$(hostname -s)
kubectl label node $(hostname -s) node-role.kubernetes.io/worker=worker
EOF
```

## Cluster OUTPUT
![Screenshot (670)](https://github.com/user-attachments/assets/ff047ced-3e1d-4af9-9f59-c00f9d6a5f64)
![Screenshot (671)](https://github.com/user-attachments/assets/5b9d86d6-cb60-4ce7-88a7-c020505cfbb9)
![Screenshot (673)](https://github.com/user-attachments/assets/a4c4dbac-d29a-4f49-9e6a-eff3f094cfa5)
![Screenshot (674)](https://github.com/user-attachments/assets/884c6e41-2225-421f-b112-743cda602c3d)



## Nginx Deployment OUTPUT
![Screenshot (672)](https://github.com/user-attachments/assets/053884c3-e5ea-4c9d-9802-954cb81a7835)
![Screenshot (678)](https://github.com/user-attachments/assets/bce8a4f3-770b-4251-b1af-197096c8aef0)
![Screenshot (681)](https://github.com/user-attachments/assets/495fad4a-63ec-465c-8ea7-946418c06d98)

## React.js Mobile Shop Application Deployment OUTPUT

![Screenshot (690)](https://github.com/user-attachments/assets/7a246322-68d0-43c2-a221-67d80a1f1a2f)
![Screenshot (687)](https://github.com/user-attachments/assets/4f1ff2ac-32ae-4b10-8988-fbfcaf184470)

## Get Kubernetes Cluster Node IP
![Screenshot (675)](https://github.com/user-attachments/assets/0b886575-7e98-435b-bb41-4ec3ae55c925)

## Application OUTPUT
![Screenshot (691)](https://github.com/user-attachments/assets/e28019d8-433c-46fe-bee8-a7a7db19ea35)
![Screenshot (692)](https://github.com/user-attachments/assets/ca5c8037-d0e3-4111-a8ad-e884918e5602)
![Screenshot (694)](https://github.com/user-attachments/assets/43cacfd9-5231-47e0-9097-32c471363f9b)

## overall OUTPUT
![Screenshot (679)](https://github.com/user-attachments/assets/4fb8248d-ad4e-4e39-9f12-a0c90327ad5a)
![Screenshot (700)](https://github.com/user-attachments/assets/e9ae4959-e585-45b8-be72-f691090d18bc)
![Screenshot (701)](https://github.com/user-attachments/assets/ff712191-3887-4909-bcc8-c1e67557d2e9)
![Screenshot (702)](https://github.com/user-attachments/assets/4249b39e-5207-48b6-8826-2bb0a7f08dae)




**This project highlights the steps to create a Kubernetes cluster using Vagrant and Git Bash, leveraging kubeadm, kubectl, kubelet, and CRI-O for cluster management and container orchestration. With Calico ensuring networking and NodePort providing external access, the deployment of the NGINX and React.js mobile shop applications showcases a practical use case for local Kubernetes development and application testing.**


# **Thank you.......**
