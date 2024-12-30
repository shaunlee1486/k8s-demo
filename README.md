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
- `kubectl apply -f .\2.rs.yaml` => create ReplicaSet

# Deployment trong Kubernetes triển khai cập nhật và scale

- Deployment quản lý một nhóm Pod - các Pod được nhân bản, nó tự động thay thế các Pod bị lỗi, không phản hồi bằng pod mới tạo ra. Như vậy, deploymnet đảm bảo ứng dụng của bạn có một (hay nhiều) Pod để phục vụ các yêu cầu.

- Deployment sử dụng mẫu Pod (Pod template -chứa định nghĩa / thiết lập về Pod) để tạo các Pod (các nhân bản replica), khi template này thay đổi, các Pod mới sẽ được tạo để thay thế Pod cũ ngay lập tức.

- Deployment tạo ra các ReplicaSet, ReplicaSet tạo ra và quản lý các pods

# Metrics Server trên Kubernetes

- metrics server trong kubernetes ([metrics server](https://github.com/kubernetes-sigs/metrics-server)) giám sát về tài nguyên sử dụng trên cluster, cung cấp các API để các thành phần khác truy vấn đến biết được và mức độ sử dụng tài nguyên (CPU, Memory) của Pod, Node ... Cần có Metric Server để HPA hoạt động chính xác

# [Service](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#replicaset-v1-apps) và Secret Tls trong Kubernetes

- Service là một resouce sẽ tạo ra một single, constant point của một nhóm Pod phía sau nó. Mỗi service sẽ có một địa chỉ IP và port không đổi, trừ khi ta xóa nó đi và tạo lại. Client sẽ mở connection tới service, và connection đó sẽ được dẫn tới một trong những Pod ở phía sau.
- Service (micro-service) trong Kubernetes là một dịch vụ mạng, tạo cơ chế cân bằng tải (load balancing) truy cập đến các điểm cuối (thường là các Pod) mà Service đó phục vụ.
- truy cập đến các pod thông qua ClusterIP Service, khi các được tạo mới quá nhiều

<image src="./image/ClusterIPService.png" />

# DaemonSet trong Kubernetes

- DaemonSet hoạt động tương tự như ReplicaSet, nghĩa là nó có thể tạo và quản lý các pod.  DaemonSet tạo ra trên mỗi Node chỉ có 1 pod 
- DaemonSet (ds) đảm bảo chạy trên mỗi NODE một bản copy của POD. Triển khai DaemonSet khi cần ở mỗi máy (Node) một POD, thường dùng cho các ứng dụng như thu thập log, tạo ổ đĩa trên mỗi Node ... Dưới đây là ví dụ về DaemonSet, nó tạo tại mỗi Node một POD chạy nginx

# Job trong Kubernetes

- Job (jobs) có chức năng tạo các POD đảm bảo nó chạy và kết thúc thành công. Khi các POD do Job tạo ra chạy và kết thúc thành công thì Job đó hoàn thành. Khi bạn xóa Job thì các Pod nó tạo cũng xóa theo. Một Job có thể tạo các Pod chạy tuần tự hoặc song song. Sử dụng Job khi muốn thi hành một vài chức năng hoàn thành xong thì dừng lại (ví dụ backup, kiểm tra ...)

Khi Job tạo Pod, Pod chưa hoàn thành nếu Pod bị xóa, lỗi Node ... nó sẽ thực hiện tạo Pod khác để thi hành tác vụ.

# CronJob trong Kubernetes

CronJob (cj) - chạy các Job theo một lịch định sẵn. Việc lên lịch cho CronJob khai báo giống Cron của Linux. [Xem Sử dụng Cron, Crontab từ động chạy script trên Server Linux](https://xuanthulab.net/su-dung-cron-crontab-tu-dong-chay-script-tren-server-linux.html)

# Persistent Volume (pv) và Persistent Volume Claim (pvc) trong Kubernetes

- Persistent Volume (pv) là một phần không gian lưu trữ dữ liệu trong cluster, các PersistentVolume giống với Volume bình thường tuy nhiên nó tồn tại độc lập vói Pod (Pod bị xóa Pv vẫn tồn tại), có nhiều loại PersistentVolume có triển khai như NFS (dịch vụ chia sẻ file), Clusterfs,.. 
- Persistent Volume Claim (pvc) là yêu cầu sử dụng không gian lưu trữ (sử dụng Pv). hình dung pv giống như Node, pvc giống như Pod. Pod chạy nó sử dụng tài nguyên của Node, pvc hoạt động nó sử dụng tài nguyên của Pv
- 1 pv chỉ có một pvc

# Ingress 

- Ingress là thành phần được dùng để điều hướng các yêu cầu traffic giao thức HTTP và HTTPS từ bên ngoài (interneet) vào các dịch vụ bên trong Cluster.

- Ingress chỉ để phục vụ các cổng, yêu cầu HTTP, HTTPS còn các loại cổng khác, giao thức khác để truy cập được từ bên ngoài thì dùng Service với kiểu NodePort và LoadBalancer

- Để Ingress hoặt động, hệ thồng cần một điều khiển ingress trước (Ingress controller), có nhiều loại để chọn sử dụng (tham khảo Ingress Controller)

- NGINX Ingress Controller for Kubernetes. HAProxy Ingress Controller
