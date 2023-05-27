# Pré requisitos
Garantir que ficheiro hosts do cluster tem os nomes e IP's dos nodes todos
Desligar SELinux
```
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
Configurar firewalld de acordo a necessidade\
Garantir que swap ta desligado e comentado em /etc/fstab:\
`swapoff -a`

# Container Runtime
## Pré requisitos
Carregar modulos e configurar sysctl:
```
sudo cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

sudo cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

Aplicar alterações no kernel:
```
sudo sysctl --system
```

Garantir que alterações foram aplicadas:
```
sudo lsmod | grep br_netfilter
sudo lsmod | grep overlay
sudo sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

Fazer download do containerd, serviço systemd e runc, extrair, carregar serviço, iniciar e enable no boot
```
sudo wget https://github.com/containerd/containerd/releases/ -P /tmp/
sudo tar Cxzvf /usr/local /tmp/containerd*
sudo wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -P /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
sudo wget https://github.com/opencontainers/runc/releases/ -P /tmp/
sudo install -m 755 /tmp/runc.amd64 /usr/local/sbin/runc
sudo wget https://github.com/containernetworking/plugins/releases/download/ -P /tmp/
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin /tmp/cni-plugins-linux-amd64*
```

Pegar nas configurações default e popular o ficheiro de configuração - alterar o `SystemdCgroup = false` para `true`:
```
sudo mkdir -p /etc/containerd
sudo containerd config default
sudo vi /etc/containerd/config.toml
sudo systemctl restart containerd
```

# Kubernetes cluster
Configurar repositorio de kubernetes
```
sudo cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```

Instalar pacotes necessarios e iniciar serviços:
```
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

Reboot
```
reboot
```

No control plane, criar a rede de pods:
```
kubeadm init --pod-network-cidr 10.10.0.0/16 --kubernetes-version 1.26.1 --node-name k8s-controlplane
```

Se o comando anterior for bem sucessido irá pedir para executar o seguinte:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

No control plane criar o pod de rede, editar custom-resources.yaml com o IP do comando `kubeadm init` e criar:
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/custom-resources.yaml
vi custom-resources.yaml
kubectl apply -f custom-resources.yaml
```

No control plane, obter o token para um node entrar no cluster
```kubeadm token create --print-join-command```

Nos worker nodes executar o output do comando anterior


Refs:\
https://github.com/containerd/containerd/releases/download/v1.6.16/containerd-1.6.16-linux-amd64.tar.gz\
https://raw.githubusercontent.com/containerd/containerd/main/containerd.service\
https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.amd64\
https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz

Ajudas:\
https://www.itsgeekhead.com/tuts/kubernetes-126-ubuntu-2204.txt\
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/\
https://kubernetes.io/docs/setup/production-environment/container-runtimes/