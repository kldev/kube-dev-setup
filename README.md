# Kubernetes for Development (NOT PRODUCTION)

Some virutualization solution like Proxmox for 3 virtual machine (1 master, 2 worker-node)

I use *Rocky Linux 8* as linux machine.

Start with 3 instalation with default server config. Setup each machine IP , enable networking.

## after machine instalation

- first setup hosts on each machine and your workstation */etc/hosts*

```
192.168.5.180 k8s.local.lab.pl
192.168.5.180 master.local.lab.pl
192.168.5.185 worker1.local.lab.pl
192.168.5.186 worker2.local.lab.pl
192.168.5.200 docker.local.lab.pl # my extra machine with docker registry
```

- disable firewall

```
sudo systemctl disable firewalld
```

- disable *swap*
```
sudo swapoff -a
```

Edit /etc/fstab (remove swamp entry)

- disable *selinux*

```
sudo setenforce 0
sudo sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```

- update kernel networking settings

```
sudo cat << EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

- install some software

```
sudo dnf install iproute-tc chrony -y
```

- reboot each nodes

- setup Docker CE ( in my setup I use docker as kuberntes contaienr runtime  )

```
sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf update
sudo mkdir /etc/docker
```
- setup docker settings and http insecure registry

```
cat <<EOF | sudo tee /etc/docker/daemon.json
{
"exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts": {
"max-size": "100m"
},
"storage-driver": "overlay2",
"insecure-registries": ["docker.local.lab.pl"]
}
EOF
```
- install docker
```
sudo dnf install docker-ce docker-ce-cli containerd.io
sudo systemctl enable docker
sudo usermod -aG docker $USER
```

- reboot all nodes

- kuberntes install (master and workers!)

```
cat << EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.
exclude=kubelet kubeadm kubectl
EOF

sudo dnf update -y

# day of my install version was 1.22.2

sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable kubelet.service
sudo systemctl start kubelet.service
```

### master setup (master.local.lab.pl)

- init node master

```
sudo kubeadm init --node-name master --pod-network-cidr 10.10.0.0/16 \
--control-plane-endpoint=k8s.local.lab.pl --apiserver-advertise-address=192.168.5.180
kubeadm token create --print-join-command
```

- install pod network

```
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

- [optional] - for development we can untaint master node to allow schedule workloads

```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

## worker 1 and worker 2 setup (join cluster)

- on each node run join command (result of **kubeadm token create --print-join-command**)
```
kubeadm join k8s.local.lab.pl:6443 --token j1tl0p.3abgsxddc2rfv2dk \
--discovery-token-ca-cert-hash sha256:a57353be3f09fcca49c1cfca7fe39d14778912da56ff2741f44bc72f4456d006
```

### Workstation steps

- copy ssh keys
```
ssh-copy-id -i ~/.ssh/id_lab.pub master.local.lab.pl
ssh-copy-id -i ~/.ssh/id_lab.pub worker1.local.lab.pl
ssh-copy-id -i ~/.ssh/id_lab.pub worker2.local.lab.pl
```
- setup ~/.ssh/config

```
Host master.local.lab.pl
   HostName 192.168.5.180
   User xd
   IdentityFile ~/.ssh/id_rsa_lab.pub
   Port 22
   
Host worker1.local.lab.pl
   HostName 192.168.5.185
   User xd
   IdentityFile ~/.ssh/id_rsa_lab.pub
   Port 22
   
Host worker2.local.lab.pl
   HostName 192.168.5.186
   User xd
   IdentityFile ~/.ssh/id_rsa_lab.pub
   Port 22   
```


### optional stepe (INGRES)
- i did use helm method
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```
- prepeare values.yml file
``` 
controller:
  hostNetwork: true
  service:
    type: NodePort
  nodePorts:
    http: 33000
    https: 33443
controller.service.httpsPort.enable: false
```
```
kubectl create namespace ingress-nginx
kubens ingress-nginx
helm upgrade ingress-nginx ingress-nginx/ingress-nginx -f values.yml --install
```


### test your cluster with command

```
kubectl cluster-info
kubeclt get nodes
```

### on each namespace when using private registry setup docker secert

```
kubectl create secret docker-registry regcred --docker-username=<USERNAME> \
--docker-password=<PASSWORD> --docker-email=<FAKE>@fake.mail --docker-server=docker.local.lab.pl
```

## Links
[https://www.oueta.com/linux/create-a-centos-8-or-rocky-linux-8-kubernetes-cluster-with-kubeadm/]
