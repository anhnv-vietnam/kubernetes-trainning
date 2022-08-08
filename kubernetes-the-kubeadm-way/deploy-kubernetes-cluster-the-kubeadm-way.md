# Kubernetes the kubeadm way - Dựng cụm Kubernetes bằng kubeadm

## Bước chuẩn bị:
1 server tối thiếu 1 core cpu, 1 GB ram để cài đặt LB (nếu có LB cứng thì không cần thực hiện)<br>
3 server tối thiểu 2 core cpu, 2 GB ram để cài đặt 3 node master của cụm (để quá trình demo đơn giản thì các server cần có kết nối đến internet).<br>
Nếu có điều kiện thì thêm 1 server tối thiểu 1 core cpu, 1 GB ram để cài worker node.<br>
Để đơn giản thì các server trong cụm có thông kết nối port 1024-65535 với nhau. (chi tiết các kết nối mà k8s sử dụng [xem ở đây](https://kubernetes.io/docs/reference/ports-and-protocols/))

## Bước 1: Set các biến môi trường (thực hiện trên các server trong cụm bằng user root. Chú ý thay ip và hostname của các server)
```
LB=172.31.24.250
IP1=172.31.14.81
HOSTNAME1=ip-172-31-14-81.ec2.internal
IP2=172.31.9.17
HOSTNAME2=ip-172-31-9-17.ec2.internal
IP3=172.31.12.168
HOSTNAME3=ip-172-31-12-168.ec2.internal
INTERNAL_IP=$(ip addr show eth0 | grep "inet " | awk '{print $2}' | cut -d / -f 1)
```

## Bước 2: Set up load balancer sử dụng haproxy (Nếu có LB cứng thì không cần thực hiện)
```
sudo yum install -y haproxy

cat <<EOF | sudo tee /etc/haproxy/haproxy.cfg
defaults
    timeout connect 10s
    timeout client 30s
    timeout server 30s
    log global
frontend kubernetes-frontend
    bind ${LB}:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server ${HOSTNAME1} ${IP1}:6443 check fall 3 rise 2
    server ${HOSTNAME2} ${IP2}:6443 check fall 3 rise 2
    server ${HOSTNAME3} ${IP3}:6443 check fall 3 rise 2
EOF

sudo systemctl restart haproxy
```

## Bước 2-8: Thực hiện trên các node trong cụm k8s
```
# Thực hiện với user root
sudo su -

# Set các biến môi trường (ở bước 1)

# Bước 2: Disable firewall (nếu có)

# Bước 3: Off swap
swapoff -a;sed -i '/swap/d' /etc/fstab

# Bước 4: Update sysctl settings for Kubernetes networking
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system

# Bước 5: Cài docker và start service
yum install -y docker
systemctl start docker

# Bước 6: Config repo.
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Bước 7: Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# Bước 8: Cài đặt kubelet, kubeadm, kubectl
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

# Bước 9: Khởi tạo node master đầu tiên
```
kubeadm init --control-plane-endpoint="${LB}:6443" --upload-certs --apiserver-advertise-address=${INTERNAL_IP} --pod-network-cidr=172.31.0.0/16
```

Sau khi lệnh khởi tạo cluster thực hiện thành công thì kết quả sẽ hiện thị ra tương tự như sau (lưu ý lưu lại các câu lệnh để phục vụ cho việc join các node còn lại vào cluster)<br>
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 172.31.24.250:6443 --token n6w2da.oaifzuls6yqudn9p \
        --discovery-token-ca-cert-hash sha256:a6e186a9b0eea46857f65207518b9805458cae7901adb61f1d9e3e0379b940d3 \
        --control-plane --certificate-key 8d013af807953bdd16ba843fef0cb93db718cfb5f3a99a5a49304d9c02129d41

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.31.24.250:6443 --token n6w2da.oaifzuls6yqudn9p \
        --discovery-token-ca-cert-hash sha256:a6e186a9b0eea46857f65207518b9805458cae7901adb61f1d9e3e0379b940d3 
```

## Bước 10: Deploy network cho cụm kubernetes (Sử dụng calico)
```
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.15/manifests/calico.yaml
```

## Bước 11: Download và cài đặt etcd utilities.
```
# Download etcd utilities.
wget "https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz"

Extract and install the etcd server and the etcdctl command line utility:
{
  tar -xvf etcd-v3.3.9-linux-amd64.tar.gz
  sudo mv etcd-v3.3.9-linux-amd64/etcd* /usr/local/bin/
}
```

## Bước 12: Kiểm tra trạng thái của etcd cluster.
```
ETCDCTL_API=3 etcdctl --endpoints ${LB}:2379 \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  member list
```