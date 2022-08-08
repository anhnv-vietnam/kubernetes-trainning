# Deploy a ETCD cluster the hard way - Dựng một cụm ETCD kiểu tay to

## Bước chuẩn bị:
3 server tối thiểu 1 core cpu, 1 GB ram (để quá trình demo đơn giản thì các server cần có kết nối đến internet).<br>Các server trong cụm có thông kết nối port 8379-8380 với nhau.
![Mô hình triển khai](https://d33wubrfki0l68.cloudfront.net/ad49fffce42d5a35ae0d0cc1186b97209d86b99c/5a6ae/images/kubeadm/kubeadm-ha-topology-external-etcd.svg)
<br>
<br>
## Bước 1: Set các biến môi trường (thực hiện trên từng server trong cụm. Chú ý thay các ip và hostname tương ứng của server)
```
IP1=172.31.26.202
HOSTNAME1=ip-172-31-26-202.ec2.internal
IP2=172.31.17.92
HOSTNAME2=ip-172-31-17-92.ec2.internal
IP3=172.31.26.94
HOSTNAME3=ip-172-31-26-94.ec2.internal

INTERNAL_IP=$(ip addr show eth0 | grep "inet " | awk '{print $2}' | cut -d / -f 1)
HOSTNAME=$(hostname)
```

## Bước 2: Generate Certificate Authority (chỉ cần thực hiện trên 1 server)
```
# Create private key for CA
openssl genrsa -out ca.key 2048

# Comment line starting with RANDFILE in /etc/ssl/openssl.cnf definition to avoid permission issues. This command may fail and it's ok.
sudo sed -i '0,/RANDFILE/{s/RANDFILE/\#&/}' /etc/ssl/openssl.cnf

# Create CSR using the private key
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr

# Self sign the csr using its own private key
openssl x509 -req -in ca.csr -signkey ca.key -CAcreateserial  -out ca.crt -days 1000
```

## Bước 3: Generate ETCD Server Certificate (chỉ cần thực hiện trên 1 server)
```
cat > openssl-etcd.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
IP.1 = ${IP1}
IP.2 = ${IP2}
IP.3 = ${IP3}
IP.4 = 127.0.0.1
EOF

openssl genrsa -out etcd-server.key 2048
openssl req -new -key etcd-server.key -subj "/CN=etcd-server" -out etcd-server.csr -config openssl-etcd.cnf
openssl x509 -req -in etcd-server.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out etcd-server.crt -extensions v3_req -extfile openssl-etcd.cnf -days 1000
```

## Bước 4: Copy các file Certificate sang tất cả các server trong cụm etcd. (Nếu có kết nối thì có thể dùng scp, còn không thì đẩy file lên thủ công)
```
for instance in ${IP2} ${IP3}; do
  scp -i "mykey.pem" ca.crt ca.key \
    etcd-server.key etcd-server.crt \
    ${instance}:~/
done
```

## Bước 5: Download và cài đặt etcd (Thực hiện trên các server trong cụm etcd).
```
# Download etcd binary
wget "https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz"

# Extract and install the etcd server and the etcdctl command line utility:
{
  tar -xvf etcd-v3.3.9-linux-amd64.tar.gz
  sudo mv etcd-v3.3.9-linux-amd64/etcd* /usr/local/bin/
}

#Configure the etcd Server
{
  sudo mkdir -p /etc/etcd /var/lib/etcd
  sudo cp ca.crt etcd-server.key etcd-server.crt /etc/etcd/
}

# Create etcd service file
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${HOSTNAME} \\
  --cert-file=/etc/etcd/etcd-server.crt \\
  --key-file=/etc/etcd/etcd-server.key \\
  --peer-cert-file=/etc/etcd/etcd-server.crt \\
  --peer-key-file=/etc/etcd/etcd-server.key \\
  --trusted-ca-file=/etc/etcd/ca.crt \\
  --peer-trusted-ca-file=/etc/etcd/ca.crt \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:8380 \\
  --listen-peer-urls https://${INTERNAL_IP}:8380 \\
  --listen-client-urls https://${INTERNAL_IP}:8379,https://127.0.0.1:8379 \\
  --advertise-client-urls https://${INTERNAL_IP}:8379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster ${HOSTNAME1}=https://${IP1}:8380,${HOSTNAME2}=https://${IP2}:8380,${HOSTNAME3}=https://${IP3}:8380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd and enable etcd service
{
  sudo systemctl daemon-reload
  sudo systemctl enable etcd
  sudo systemctl start etcd
  sudo systemctl status etcd
}
```

## Bước 6: Kiểm tra trạng thái etcd cluster.
```
export ETCDCTL_API=3
etcdctl member list   --endpoints=https://127.0.0.1:8379   --cacert=/etc/etcd/ca.crt   --cert=/etc/etcd/etcd-server.crt   --key=/etc/etcd/etcd-server.key
```
