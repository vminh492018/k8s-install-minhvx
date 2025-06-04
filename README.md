# Hướng Dẫn Cài Đặt Kubernetes Cluster trên RHEL 9/CentOS 9

## Mục Lục
- [Yêu cầu trước khi cài đặt](#yêu-cầu-trước-khi-cài-đặt)
- [Các bước cài đặt Kubernetes Cluster trên RHEL 9/CentOS 9](#các-bước-cài-đặt-kubernetes-cluster-trên-rhel-9centos-9)
  - [Bước 1: Cài đặt Kernel Headers](#bước-1-cài-đặt-kernel-headers)
  - [Bước 2: Nạp các Kernel Module cần thiết](#bước-2-nạp-các-kernel-module-cần-thiết)
  - [Bước 3: Cấu hình Sysctl](#bước-3-cấu-hình-sysctl)
  - [Bước 4: Tắt Swap](#bước-4-tắt-swap)
  - [Bước 5: Cài đặt Containerd](#bước-5-cài-đặt-containerd)
  - [Bước 6: Mở Firewall cho các port của Kubernetes](#bước-6-mở-firewall-cho-các-port-của-kubernetes)
  - [Bước 7: Cài đặt các thành phần Kubernetes](#bước-7-cài-đặt-các-thành-phần-kubernetes)
- [Cấu hình Master Node](#cấu-hình-master-node)
  - [Bước 8: Khởi tạo Control Plane](#bước-8-khởi-tạo-control-plane)
  - [Bước 9: Thêm các Node Worker](#bước-9-thêm-các-node-worker)
- [Mô hình triển khai nhiều Control Plane (HA) với 3 Master (Worker)](#mô-hình-triển-khai-nhiều-control-plane-ha-với-3-master-worker)
- [Triển khai NGINX test](#triển-khai-nginx-test)
- [Các phương pháp khác để expose service ra bên ngoài](#các-phương-pháp-khác-để-expose-service-ra-bên-ngoài)
- [Kết luận](#kết-luận)

---

## Yêu cầu trước khi cài đặt
- Tối thiểu 3 máy: 1 master và 2 worker node, chạy RHEL 9 hoặc CentOS 9.
- Mỗi máy có ít nhất 2GB RAM và 2 CPU core.
- Nếu không có DNS, cấu hình `/etc/hosts` trên mỗi node như sau:
  ```
  192.168.1.26  master.minhvx.k8s
  192.168.1.27  worker1.minhvx.k8s
  192.168.1.28  worker2.minhvx.k8s
  ```
- Thông tin demo:
  | Hostname            | RAM | Cores | OS                                   |
  |---------------------|-----|-------|--------------------------------------|
  | master.minhvx.k8s   | 4   | 2     | Red Hat Enterprise Linux release 9.3 |
  | worker1.minhvx.k8s  | 4   | 2     | Red Hat Enterprise Linux release 9.3 |
  | worker2.minhvx.k8s  | 4   | 2     | Red Hat Enterprise Linux release 9.3 |

---

## Các bước cài đặt Kubernetes Cluster trên RHEL 9/CentOS 9

### Bước 1: Cài đặt Kernel Headers
**Lý do:** Kernel headers cần thiết cho việc build các module kernel mà Kubernetes sử dụng.

```bash
sudo dnf clean all
sudo dnf makecache
sudo dnf install kernel-devel-$(uname -r)
```

### Bước 2: Nạp các Kernel Module cần thiết
**Lý do:** Các module này hỗ trợ networking và cân bằng tải trong Kubernetes.

```bash
sudo modprobe br_netfilter
sudo modprobe ip_vs
sudo modprobe ip_vs_rr
sudo modprobe ip_vs_wrr
sudo modprobe ip_vs_sh
sudo modprobe overlay
```
**Lưu ý:** Để các module này tự động nạp khi khởi động lại, tạo file:
```bash
cat > /etc/modules-load.d/kubernetes.conf << EOF
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
overlay
EOF
```

### Bước 3: Cấu hình Sysctl
**Lý do:** Đảm bảo kernel cho phép forwarding và xử lý traffic mạng cho container.

```bash
cat > /etc/sysctl.d/kubernetes.conf << EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```

### Bước 4: Tắt Swap
**Lý do:** Kubernetes yêu cầu tắt swap để đảm bảo hiệu năng và tính nhất quán về resource.

```bash
sudo swapoff -a
sudo sed -e '/swap/s/^/#/g' -i /etc/fstab
```

### Bước 5: Cài đặt Containerd
**Lý do:** Containerd là container runtime dùng để chạy các container cho Kubernetes.

**Thêm repo Docker CE:**
```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
**Cập nhật cache gói:**
```bash
sudo dnf makecache
```
**Cài đặt containerd:**
```bash
sudo dnf -y install containerd.io
```
**Tạo file cấu hình mặc định cho containerd:**
```bash
sudo sh -c "containerd config default > /etc/containerd/config.toml"
```
**Bật SystemdCgroup để tương thích với systemd:**
```bash
sudo vim /etc/containerd/config.toml
# Tìm dòng SystemdCgroup, sửa thành:
SystemdCgroup = true
```
**Khởi động và enable containerd:**
```bash
sudo systemctl enable --now containerd.service
sudo systemctl status containerd.service
```
**Reboot máy để áp dụng toàn bộ cấu hình:**
```bash
sudo systemctl reboot
```

### Bước 6: Mở Firewall cho các port của Kubernetes
**Lý do:** Các port này cần thiết để các thành phần của Kubernetes giao tiếp với nhau.

```bash
sudo firewall-cmd --zone=public --permanent --add-port=6443/tcp
sudo firewall-cmd --zone=public --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --zone=public --permanent --add-port=10250/tcp
sudo firewall-cmd --zone=public --permanent --add-port=10251/tcp
sudo firewall-cmd --zone=public --permanent --add-port=10252/tcp
sudo firewall-cmd --zone=public --permanent --add-port=10255/tcp
sudo firewall-cmd --zone=public --permanent --add-port=5473/tcp
sudo firewall-cmd --reload
```

### Bước 7: Cài đặt các thành phần Kubernetes
**Lý do:** kubelet (quản lý node), kubeadm (khởi tạo/tham gia cluster), kubectl (quản lý cluster).

**Thêm repo Kubernetes:**
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```
**Cài đặt các gói:**
```bash
sudo dnf makecache
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```
**Bật kubelet:**
```bash
sudo systemctl enable --now kubelet.service
```
> *Lưu ý: Nếu kubelet báo lỗi lúc này cũng không sao, sau khi join node sẽ tự động hoạt động.*

---

## Cấu hình Master Node

### Bước 8: Khởi tạo Control Plane
**Lý do:** Đây là bước tạo node quản trị trung tâm cho cụm Kubernetes.

**Tải trước các image cần thiết:**
```bash
sudo kubeadm config images pull
```
**Khởi tạo control plane (chọn CIDR phù hợp, ví dụ 10.244.0.0/16 cho Calico):**
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
**Cấu hình file kubeconfig để dùng kubectl:**
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Triển khai pod network (ví dụ Calico):**
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml
sed -i 's/cidr: 192\.168\.0\.0\/16/cidr: 10.244.0.0\/16/g' custom-resources.yaml
kubectl create -f custom-resources.yaml
```

### Bước 9: Thêm các Node Worker
**Lý do:** Kết nối các node worker vào cluster để chạy workload.

**Trên node master, lấy lệnh join:**
```bash
sudo kubeadm token create --print-join-command
```
**Chạy lệnh join trên từng node worker (thay tham số cho đúng):**
```bash
sudo kubeadm join <MASTER_IP>:<MASTER_PORT> --token <TOKEN> --discovery-token-ca-cert-hash <HASH>
```
**Kiểm tra trạng thái node:**
```bash
kubectl get nodes
```
Các node worker phải có trạng thái "Ready".

---

## Mô hình triển khai nhiều Control Plane (HA) với 3 Master (Worker)

Mô hình này giúp tăng tính sẵn sàng cho cluster khi có nhiều node master cùng quản lý control plane. Trong ví dụ này, cả 3 node đều có thể chạy workload (vừa làm master vừa làm worker).

Giả sử các node:
- master.minhvx.k8s (192.168.1.26)
- worker1.minhvx.k8s (192.168.1.27)
- worker2.minhvx.k8s (192.168.1.28)

### Bước 1: Chuẩn bị hosts file trên cả 3 node (nếu không có DNS)
```bash
cat <<EOF | sudo tee -a /etc/hosts
192.168.1.26  master.minhvx.k8s
192.168.1.27  worker1.minhvx.k8s
192.168.1.28  worker2.minhvx.k8s
EOF
```

### Bước 2: Khởi tạo control plane trên master đầu tiên (master.minhvx.k8s)

> Lưu ý: Bạn nên dùng một Virtual IP hoặc DNS cho endpoint control-plane nếu triển khai thực tế HA với load balancer, ví dụ: --control-plane-endpoint "master.minhvx.k8s:6443"
> Trong ví dụ này, giả định master.minhvx.k8s là endpoint.

```bash
sudo kubeadm init --control-plane-endpoint "master.minhvx.k8s:6443" --upload-certs --pod-network-cidr=10.244.0.0/16
```

Sau khi init thành công, làm các bước sau trên master.minhvx.k8s:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Triển khai Calico network:
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml
sed -i 's/cidr: 192\.168\.0\.0\/16/cidr: 10.244.0.0\/16/g' custom-resources.yaml
kubectl apply -f custom-resources.yaml
```

### Bước 3: Thêm master thứ 2 và 3 vào control plane

Trên master.minhvx.k8s, lấy lệnh join cho control-plane:
```bash
kubeadm token create --print-join-command --certificate-key $(kubeadm init phase upload-certs --upload-certs | tail -1)
```
Copy lệnh join này (có cả --control-plane và --certificate-key) và chạy trên worker1.minhvx.k8s và worker2.minhvx.k8s:

```bash
sudo kubeadm join master.minhvx.k8s:6443 --token <TOKEN> --discovery-token-ca-cert-hash <HASH> --control-plane --certificate-key <CERT_KEY>
```

Trên từng node master mới join, cấu hình kubeconfig:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Bước 4: Cho phép cả 3 master chạy workload (bỏ taint mặc định)

Mặc định các node master sẽ bị taint để không chạy workload. Để các node này cũng là worker, hãy bỏ taint này:

```bash
kubectl taint nodes master.minhvx.k8s node-role.kubernetes.io/control-plane:NoSchedule-
kubectl taint nodes worker1.minhvx.k8s node-role.kubernetes.io/control-plane:NoSchedule-
kubectl taint nodes worker2.minhvx.k8s node-role.kubernetes.io/control-plane:NoSchedule-
```

### Bước 5: Kiểm tra trạng thái cluster

```bash
kubectl get nodes
```
Tất cả node phải ở trạng thái "Ready" và đều có vai trò control-plane.
---

## Triển khai NGINX test
**Lý do:** Kiểm tra cluster hoạt động ổn định.

**Tạo file `nginx-deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```
**Triển khai:**
```bash
kubectl apply -f nginx-deployment.yaml
kubectl get deployments
kubectl get pods
```
**Expose service ra ngoài:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```
```bash
kubectl apply -f nginx-service.yaml
kubectl get service nginx-service
```
Truy cập vào external IP của service để kiểm tra.

---

## Các phương pháp khác để expose service ra bên ngoài

| Cách         | Mô tả |
|--------------|-------|
| NodePort     | Mở port trên tất cả node, NAT/Port Forward từ router về node. |
| ExternalIPs  | Gán IP ngoài vào trường externalIPs trong manifest Service, cấu hình router forward về node. |
| Ingress      | Dùng Ingress controller với NodePort, forward port/traffic từ router tới NodePort của Ingress. |
| HostNetwork  | Pod dùng trực tiếp network của node, bind luôn IP node. |

Tùy vào setup mạng và bảo mật mà chọn phương án phù hợp.

---
