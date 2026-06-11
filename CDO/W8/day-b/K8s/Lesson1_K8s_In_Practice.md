# Kubernetes in Practice: Kiến thức cốt lõi và Thực hành

Tài liệu này tổng hợp các khái niệm quan trọng nhất trong Kubernetes (K8s) dành cho người mới bắt đầu. Bao gồm bức tranh tổng thể, các ví dụ thực tế cực kỳ dễ hiểu và các lệnh thực hành cốt lõi.

---

## 1. Bức tranh lớn: Kubernetes là gì và giải quyết vấn đề gì?

Docker giúp bạn quản lý Container trên 1 máy tính. Nhưng khi hệ thống lớn lên (hàng nghìn container trên hàng chục máy chủ), bạn sẽ đối mặt với các vấn đề đau đầu:
- Container bị chết làm sao tự khởi động lại?
- Đột ngột có 10.000 user vào thì nhân bản app lên thế nào?
- Khi nhân bản lên, làm sao để chia đều request cho các app?
- Dữ liệu Database lưu ở đâu để container chết không bị mất?
=> **Kubernetes (K8s) chính là "hệ điều hành" sinh ra để giải quyết tất cả những việc đó một cách hoàn toàn tự động.**

### Sơ đồ luồng đi của một Request (Cần thuộc lòng)
Đây là sơ đồ bạn sẽ gặp trong 99% các hệ thống K8s thực tế, hãy nhớ đúng thứ tự này:
```text
Người dùng
    ↓
Ingress         (Điều hướng theo Tên miền / Đường dẫn)
    ↓
Service         (Điểm truy cập cố định & Cân bằng tải)
    ↓
Deployment      (Đảm bảo luôn có đủ số lượng Pod)
    ↓
Pod             (Ngôi nhà chứa Container)
    ↓
Container       (Nơi chứa code thực sự)
    │
    ├── Đọc cấu hình từ ConfigMap / Secret
    └── Ghi dữ liệu vào ổ cứng qua PVC / PV
```

---

## 2. Giải ngố 10 khái niệm K8s bằng ví dụ đời thực

Đừng học vẹt các file YAML ngay từ đầu, hãy hiểu bản chất qua các ví dụ đời thường sau:

### 2.1. Pod & Container
- **Vấn đề:** Container chạy độc lập, nhưng nhiều lúc 2 container cần dùng chung ổ đĩa và mạng (Ví dụ: 1 app chính và 1 app thu thập log). K8s tạo ra **Pod** để bọc chúng lại.
- **Định nghĩa:** K8s không quản lý Container trực tiếp, nó quản lý **Pod**. Pod là đơn vị nhỏ nhất.
- **Ví dụ:** **Pod là Ngôi nhà**, còn **Container là Người sống trong nhà**. K8s chỉ quản lý và cấp nhà, chứ không rảnh đi đếm từng người.

### 2.2. Deployment
- **Định nghĩa:** Là thứ giúp quản lý số lượng Pod cho các app không lưu dữ liệu cố định (Stateless như Web, API).
- **Ví dụ:** Deployment là **Người quản lý quán cà phê**. Quán quy định phải có 3 nhân viên phục vụ (`replicas: 3`). Nếu 1 nhân viên nghỉ việc (Pod chết), quản lý lập tức tuyển ngay 1 người mới để luôn giữ đúng số lượng là 3 *(Cơ chế Self-healing)*. 
- *(Bên dưới Deployment sử dụng một cơ chế gọi là ReplicaSet để thực thi việc này).*

### 2.3. Service
- **Định nghĩa:** Pod rất dễ chết và sinh ra với IP mới liên tục. **Service** đóng vai trò là **bộ định tuyến nội bộ của K8s**, có nhiệm vụ nhận và điều hướng traffic đến các Pod. Nó cung cấp một IP và Tên (DNS) cố định đứng trước nhóm Pod đó, đồng thời làm nhiệm vụ Cân bằng tải (Load Balancing).
- **Ví dụ:** Frontend muốn gọi Backend nhưng IP của Backend đổi liên tục. Service chính là **Tổng đài nội bộ**. Frontend chỉ cần gọi đúng tên tổng đài (`http://backend-service`), tổng đài sẽ tự động điều hướng cuộc gọi (request) đến các Pod Backend nào đang rảnh.

### 2.4. Ingress (Layer 7)
- **Định nghĩa:** Điều hướng traffic từ Internet vào các Service bên trong cụm dựa trên Tên miền (Domain) hoặc Đường dẫn (Path).
- **Ví dụ:** Ingress là **Lễ tân toà nhà**. 
  - Khách vào nói: "Tôi đến phòng API" (`shop.com/api`) -> Lễ tân chỉ đường sang Service API.
  - Khách nói: "Tôi đến phòng Web" (`shop.com/web`) -> Lễ tân dẫn sang Service Web.

### 2.5. ConfigMap & Secret
- **ConfigMap:** Là **Tờ giấy ghi chú cấu hình** (ví dụ URL database, Port). App sẽ đọc tờ giấy này để chạy thay vì ghi cứng vào code. Đổi môi trường cực kỳ dễ dàng.
- **Secret:** Là **Két sắt**. Giống hệt ConfigMap nhưng chuyên dùng để lưu Password, Token, API Key (để mã hoá, không lộ trong source code).

### 2.6. PV và PVC (Lưu trữ dữ liệu vĩnh viễn)
- **Vấn đề:** Ổ cứng trong Pod là tạm thời. Pod chết = Mất trắng dữ liệu (rất nguy hiểm cho Database).
- **PV (Persistent Volume):** Là ổ cứng vật lý thật sự (VD: AWS EBS 100GB). Được ví như **Kho chứa hàng**.
- **PVC (Persistent Volume Claim):** Ứng dụng không quan tâm ổ cứng là loại gì, nó chỉ viết một **Đơn xin cấp ổ cứng** (VD: "Cho tôi 20GB"). PVC chính là **Phiếu yêu cầu xuất kho**. K8s sẽ tự lấy Phiếu này đi tìm Kho (PV) phù hợp để gắn vào.

### 2.7. HPA (Horizontal Pod Autoscaler)
- **Định nghĩa:** Tự động tăng/giảm số lượng Pod dựa trên tải (CPU/RAM).
- **Ví dụ:** Quán đông khách đột xuất (Flash Sale, CPU > 70%) -> HPA tự động thuê thêm nhân viên (Tự động tăng Pod từ 3 lên 10). Tối hết khách -> HPA tự cho nghỉ bớt để tiết kiệm tiền (Giảm về 3).

### 2.8. StatefulSet & DaemonSet (Các controller đặc thù)
- **StatefulSet:** Dành cho Database (MySQL, Kafka). Nó cấp cho mỗi Pod một tên cố định (`mysql-0`, `mysql-1`) và ổ cứng cố định không đổi. Nếu `mysql-0` chết, K8s tạo lại Pod mới vẫn mang tên `mysql-0` và gắn lại đúng ổ đĩa cũ.
- **DaemonSet:** Đảm bảo **mỗi Node (máy chủ) phải có đúng 1 Pod**.
  - Ví dụ: Lắp **Camera an ninh** - mỗi tầng của toà nhà (Node) đều bắt buộc phải lắp đúng 1 cái camera. Thêm tầng mới tự có camera mới.
- **Fluent Bit (Thường chạy bằng DaemonSet):** Là **Nhân viên thu thư**. Đi thu gom các file log rải rác từ mọi Pod trên máy chủ, gom lại và gửi về trung tâm (như Elasticsearch, OpenSearch).

---

## 3. Các lưu ý chuyên sâu về Vận Hành

- **Service có 3 loại chế độ hoạt động:**
  - **1. Chế độ ClusterIP (Mặc định)**
    - **Cách hoạt động:** Lễ tân chỉ trực mạng lưới điện thoại nội bộ.
    - **Ý nghĩa:** Các Pod (căn hộ) bên trong K8s gọi cho nhau thì lễ tân sẽ nối máy. Nhưng nếu người từ bên ngoài (Internet, ALB) cố gắng gọi vào thì sẽ bị từ chối hoàn toàn. Nó hoàn toàn cô lập với thế giới bên ngoài.
  - **2. Chế độ NodePort (Cái bạn đang dùng)**
    - **Cách hoạt động:** Bạn ra lệnh cho lễ tân: "Hãy kéo một đường dây nóng ra ngay mép tường ngoài cùng của tòa nhà (chính là con máy ảo EC2) và đặt ở đó một cái cổng có số là 30000".
    - **Ý nghĩa:** Lúc này, Service vẫn làm nhiệm vụ điều hướng nội bộ, nhưng nó được cấp thêm siêu năng lực là "thò" một cái cổng ra hẳn thế giới bên ngoài. Bất kỳ ai ở bên ngoài (như AWS ALB) chỉ cần đập cửa đúng số 30000 của con EC2, Service sẽ mở cửa đón lấy dữ liệu và dắt vào tận tay chiếc Pod Nginx đang nằm bên trong.
  - **3. Chế độ LoadBalancer (Dùng khi đi làm thực tế)**
    - **Cách hoạt động:** Lễ tân tự động gọi điện lên tổng đài của AWS và bảo: "Hãy tự động tạo cho tôi một cái Load Balancer thật xịn ở ngoài đám mây rồi nối thẳng vào đây cho tôi".
    - **Ý nghĩa:** Chế độ này thường dùng khi bạn dùng EKS. Còn trong bài Lab này, vì bạn dùng Terraform để tự tay xây Load Balancer (ALB), nên bạn không dùng chế độ này của K8s mà dùng chế độ NodePort để thủ công nối ALB vào K8s.
- **Probes (Kiểm tra sức khoẻ - Cực quan trọng):**
  - `livenessProbe` ("Mày còn sống không?"): Bị Treo/Deadlock -> K8s sẽ **Restart** container.
  - `readinessProbe` ("Mày sẵn sàng nhận khách chưa?"): Khởi động chậm/Quá tải -> K8s tạm thời ngưng đẩy traffic vào Pod này (Nhưng **không restart**). Giúp app không bao giờ trả về lỗi 502 cho khách.
- **Requests & Limits (Tài nguyên):**
  - `requests`: Lượng CPU/RAM tối thiểu K8s phải giữ chỗ cho Pod.
  - `limits`: Trần tối đa. Quá giới hạn CPU thì bị làm chậm (throttle), quá giới hạn RAM thì bị K8s bắn bỏ lập tức (`OOMKilled`).
- **Cập nhật không Downtime:**
  - K8s dùng cơ chế **Rolling Update**: Khởi động Pod mới, chờ nó báo "Ready", rồi mới từ từ tắt Pod cũ.
  - Nếu bản update bị lỗi, có thể dùng `rollout undo` để quay xe lùi phiên bản ngay lập tức.

---

## 4. Cheat Sheet Thực Hành (Hands-on Lab)

Dưới đây là các lệnh gõ hằng ngày. Thay vì học thuộc lý thuyết, hãy cài `minikube` và gõ thử các lệnh này:

```bash
# 1. Khởi động cụm K8s local trên máy tính
minikube start 
# hoặc dùng kind: kind create cluster --name lab

# 2. Triển khai (Deploy) app Nginx với 3 bản sao
kubectl create deployment web --image=nginx:1.27 --replicas=3

# 3. Theo dõi tiến trình tạo Pod (Bấm Ctrl+C để thoát)
kubectl get pods -w

# 4. Xem phả hệ Deployment -> ReplicaSet -> Pod
kubectl get deploy,rs,pods

# 5. Mở Service để lộ app ra bên ngoài (Dùng loại NodePort)
kubectl expose deployment web --type=NodePort --port=80

# 6. Mở trình duyệt xem app thực tế
minikube service web --url
# Hoặc Forward Port thủ công:
kubectl port-forward svc/web 8080:80  # Mở localhost:8080 trên trình duyệt

# 7. Thử nghiệm Scale
kubectl scale deploy/web --replicas=5

# 8. Thử nghiệm Self-healing (Tự phục hồi)
# Xoá thử 1 Pod, lập tức 1 Pod mới sẽ được sinh ra
kubectl delete pod <tên-pod>

# 9. Rolling Update (Cập nhật phiên bản mới)
kubectl set image deploy/web web=nginx:1.28
kubectl rollout status deploy/web

# 10. Rollback (Lỗi quá, quay xe về bản cũ)
kubectl rollout undo deploy/web

# 11. Debug khi Pod lỗi (VD: ImagePullBackOff do sai tên image)
kubectl describe pod <tên-pod>

# 12. Dọn dẹp máy sạch sẽ
kubectl delete deploy/web svc/web
minikube delete
```

> **💡 Lời khuyên cuối cùng:** Học K8s đừng học thuộc file YAML. Hãy chạy `minikube` lên và tự đặt câu hỏi: *"Nếu không có cái này thì vấn đề gì sẽ xảy ra?"* (Ví dụ: Không có Service thì Frontend gọi bằng gì? Không có PVC thì database lưu ở đâu?). Đó là cách hiểu Kubernetes sâu sắc nhất!
