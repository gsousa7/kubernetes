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


# Container Runtime - todos os nodes
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
sudo wget https://github.com/containerd/containerd/releases/download/v1.7.1/containerd-1.7.1-linux-amd64.tar.gz -P /tmp/
sudo tar Cxzvf /usr/local /tmp/containerd*

sudo wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -P /etc/systemd/system/

sudo systemctl daemon-reload
sudo systemctl enable --now containerd

sudo wget https://github.com/opencontainers/runc/releases/download/v1.1.7/runc.amd64 -P /tmp/
sudo install -m 755 /tmp/runc.amd64 /usr/local/sbin/runc

sudo wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz -P /tmp/
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


# Kubernetes cluster - apenas em control plane
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

Criar a rede de pods:
```
kubeadm init --pod-network-cidr 10.10.0.0/16 --kubernetes-version 1.26.1 --node-name k8s-controlplane
```

Se o comando anterior for bem sucessido irá pedir para executar o seguinte:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Criar o rede dos pods com o Calico, editar custom-resources.yaml com o IP do comando `kubeadm init` e criar:
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/custom-resources.yaml
vi custom-resources.yaml
kubectl apply -f custom-resources.yaml
```

Obter o token para um node entrar no cluster
```kubeadm token create --print-join-command```


# Worker Nodes
Executar o output do comando anterior.




---
# Info
## Doc de 3rd parties:
[containerd doc](https://github.com/containerd/containerd/)\
[containerd releases](https://github.com/containerd/containerd/releases/)\
[runc doc](https://github.com/opencontainers/runc/)\
[runc releases](https://github.com/opencontainers/runc/releases/)\
[CNI doc](https://github.com/containernetworking/plugins/)\
[CNI releases](https://github.com/containernetworking/plugins/releases/)\
[Calico](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)

## Ajudas:
[Ajuda completa - está em ubuntu](https://www.itsgeekhead.com/tuts/kubernetes-126-ubuntu-2204.txt)\
[Kubernetes doc - Kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)\
[Kubernetes doc - containerd](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)