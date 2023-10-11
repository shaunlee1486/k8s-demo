https://xuanthulab.net/gioi-thieu-va-cai-dat-kubernetes-cluster.html

- tạo virtualbox guest VM bằng vagrant
- cài đặt docker lên guestVM bằng Ansible
- tạo cụm khả dụng cao Kubernetes bằng subeadm và Asible

# master machine in virtualbox

- Tạo node trên virtualbox thông qua vagrantfile

## Khởi tạo Cluster

Trong lệnh khởi tạo cluster có tham số `--pod-network-cidr` để chọn cấu hình mạng của POD, do dự định dùng [Addon](https://kubernetes.io/docs/concepts/cluster-administration/addons/) [calico](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart#install-calico) nên chọn --pod-network-cidr=192.168.0.0/16

Gõ lệnh sau để khở tạo là nút master của Cluster
`kubeadm init --apiserver-advertise-address=172.16.10.100 --pod-network-cidr=192.168.0.0/16`

fix [ERROR CRI]: container runtime is not running

```
https://forum.linuxfoundation.org/discussion/862825/kubeadm-init-error-cri-v1-runtime-api-is-not-implemented
yum update -y
yum remove containerd
yum install containerd.io -y
yum update -y
rm /etc/containerd/config.toml
systemctl restart containerd
```

1. kubeadm init --apiserver-advertise-address=172.16.10.100 --pod-network-cidr=192.168.0.0/16
2. mkdir -p $HOME/.kube
3. sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
4. sudo chown $(id -u):$(id -g) $HOME/.kube/config
5. kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

Kết nối Node vào Cluster
`kubeadm token create --print-join-command`
=> `kubeadm join 172.16.10.100:6443 --token zxi6kf.ww4d7xaueu4oxhhm --discovery-token-ca-cert-hash sha256:18774d828737ffef74ee3f6223d7f8af94eaa62087dab90da1499d6bf52b8327`

# worker connect to cluster

node: ping 172.16.10.100 if timeout
run `kubeadm join 172.16.10.100:6443 --token zxi6kf.ww4d7xaueu4oxhhm --discovery-token-ca-cert-hash sha256:18774d828737ffef74ee3f6223d7f8af94eaa62087dab90da1499d6bf52b8327`

fix ERROR FileContent--proc-sys-net-bridge-bridge-nf-call-iptables]: /proc/sys/net/bridge/bridge-nf-call-iptables does not exist

```
https://stackoverflow.com/questions/44125020/cant-install-kubernetes-on-vagrant

modprobe br_netfilter
echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
echo 1 > /proc/sys/net/bridge/bridge-nf-call-ip6tables
sysctl -p
```

fix [ERROR FileContent--proc-sys-net-ipv4-ip_forward]: /proc/sys/net/ipv4/ip_forward contents are not set to 1

```
https://stackoverflow.com/questions/55531834/kubeadm-fails-to-initialize-when-kubeadm-init-is-called
echo 1 > /proc/sys/net/ipv4/ip_forward

```

# C:\Users\Admin\.kube

kubectl trên máy host, kết nối đến 3 máy trên virtual
lấy file config master trên virtual về máy host: scp root@172.16.10.100:/etc/kubernetes/admin.conf ./config-mycluster

merge file config-mycluster -> config
hiện có 2 context: docker-desktop vs kubernetes-admin@kubernetes
chuyển sang dùng context: kubernetes-admin@kubernetes
`kubectl config use-context kubernetes-admin@kubernetes`

# Tạo pod từ file .yaml

- p: create pod nhanh
- d: create deployment
  [Pod v1 core](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#pod-v1-core)
  1 core cpu = 1000m

`kubectl apply -f .\1-swarmtest-node.yaml` => create pod

## command quản trị pod

1. get all pod

- `kubectl get pod -o wide`

2. delete pod

- `kubectl delete pod <name-pod>`, xóa pod có tên ungdungnode: `kubectl delete pod ungdungnode`
- `kubectl delete -f <file-yaml-create-pod>`, xóa pod có tên file 1-swarmtest-node: `kubectl delete -f 1-swarmtest-node`

3. xem chi tiết pod

- `kubectl describe pod/<name-pod>`, xem chi tết pod có tên nginxapp: `kubectl describe pod/nginxapp`

4. hiển thị Manifests của pod

- `kubectl get pod/ungdungnode -o yaml`

5. sửa Manifests

- `kubectl edit pod/ungdungnode`

6. xem log của pod

- `kubectl logs -f pod/tools`

7. thi hành lệnh trong pod

- `kubectl exec ungdungnode ls /`

8. truy cập terminal của pod có tên tools

- pod có 1 container: `kubectl exec -it tools -- bash`
- pod có nhiều container `kubectl exec -it tools -c <name-container> bash`

## pod có nhiều container

- định nghĩa file create pod `4-nginx-swamtest.yaml` có nhiều container

## pod có tạo volume

- `5-nginx-swamtest-vol.yaml`

# ReplicaSet trong Kubernetes

- [ReplicaSet](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#replicaset-v1-apps) là một điều khiển Controller - nó đảm bảo ổn định các nhân bản (số lượng và tình trạng của POD, replica) khi đang chạy.

#
