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
