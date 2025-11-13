# 🧠 TƯ DUY DEVOPS (Thiết kế & Trình bày Workflow)

![DevOps Workflow](https://github.com/user-attachments/assets/2eab57f7-468a-4101-b4bd-7a84b2de1da9)

---

## 🌐 Infra-provision (Terraform: AWS Infra Setup)

Stage **Infra-provision** chịu trách nhiệm **tự động triển khai hạ tầng AWS** bằng công cụ **Terraform**.

### 🎯 Mục tiêu
Tạo toàn bộ hạ tầng cần thiết cho hệ thống DevOps bao gồm:
- **VPC**, **subnet**, **security group**, và các **EC2 instance**.
- **4 máy ảo EC2** được khởi tạo tự động:
  - 🖥️ 1 máy dùng để cài đặt và chạy **Jenkins Server** (đảm nhiệm CD pipeline).
  - ☸️ 3 máy còn lại cấu hình thành **Kubernetes Cluster** (1 master node + 2 worker nodes).

### ⚙️ Cấu hình
- Quản lý thông số (IP, SSH key, instance type, AMI, ...) bằng **variables.tf** giúp dễ mở rộng.
- **Terraform output** trả về thông tin:
  - Public IP Jenkins
  - IP các node trong cluster

### ✅ Kết quả Stage
Toàn bộ môi trường AWS được khởi tạo tự động, sẵn sàng cho Ansible tiến hành cài đặt Jenkins và thiết lập Kubernetes cluster.

**Repository:** [infra-provision](https://github.com/petclinic-devops/infra-provision)

---

## ⚙️ Infra-config (Ansible: setup Cluster k8s and Jenkins on EC2)

Stage **Infra-config** sử dụng **Ansible** để tự động cấu hình các máy EC2 được Terraform tạo ra.

### 🎯 Mục tiêu
Thiết lập môi trường triển khai bao gồm **Jenkins Server** và **Kubernetes Cluster**.

### 🔧 Quy trình
- Kết nối tới 4 EC2 instance thông qua SSH (private key từ Terraform output).
- **Máy Jenkins:**
  - Cài đặt OpenJDK, Jenkins, Docker, kubectl.
  - Cấu hình Jenkins kết nối GitHub (webhook) để trigger CD pipeline.
  - Tạo các credentials: DockerHub, GitHub Token, kubeconfig.
- **3 máy Kubernetes Cluster:**
  - Thiết lập 1 master + 2 worker.
  - Dùng `kubeadm` khởi tạo cluster.
  - Cấu hình CNI plugin (Calico / Flannel).
  - Cài đặt tiện ích: `kubectl`, `helm`.

### ✅ Kết quả Stage
- Kubernetes Cluster đã sẵn sàng để triển khai ứng dụng.
- Jenkins Server sẵn sàng thực hiện CD pipeline.

**Repository:** [infra-config](https://github.com/petclinic-devops/infra-config)

---

## 🧩 petclinic-app (App code)

`petclinic-app` là **dự án microservices mẫu** cho toàn bộ quy trình DevOps.

### ⚙️ Cấu trúc & Quy trình CI
- Dự án gồm nhiều service độc lập:
  - `customers-service`, `vets-service`, `visits-service`, `api-gateway`, ...
- Mỗi service có **Dockerfile riêng** và được build – deploy tách biệt.

### 🔄 Quy trình CI (GitHub Actions)
1. Developer push code lên GitHub.
2. Khi có thay đổi ở nhánh `main` hoặc `devops-final`, **GitHub Actions** tự động:
   - Pull source mới nhất.
   - Build từng service (Maven/Gradle).
   - Dockerize từng service.
   - Gửi thông báo trạng thái CI về Slack.
   - Push image lên **DockerHub** với version tag tương ứng.
3. Jenkins sẽ dùng các image này trong giai đoạn **CD pipeline**, triển khai lên Kubernetes qua **Helm chart**.

### ✅ Kết quả Stage
Tất cả các service được build, test, đóng gói, và lưu trữ trên DockerHub – sẵn sàng cho CD.

![CI Workflow](https://github.com/user-attachments/assets/c25b6ace-257e-45c5-8abf-9a0e28e7cba0)
![DockerHub Push](https://github.com/user-attachments/assets/06601b71-677b-4530-89e2-c0fc4c68b92b)

**Repository:** [petclinic-app](https://github.com/petclinic-devops/petclinic-app)

---

## 🚀 Continuous Deployment (CD) với Jenkins

Giai đoạn **Continuous Deployment (CD)** được đảm nhiệm bởi **Jenkins**, triển khai toàn bộ các service lên Kubernetes Cluster.

### 🔧 Quy trình
- Jenkins được cài trên EC2 riêng, tích hợp webhook với GitHub.
- Khi CI hoàn tất và Docker image mới được push lên DockerHub:
  - Jenkins nhận trigger tự động hoặc được kích hoạt thủ công.
  - Pipeline (Jenkinsfile) gồm các stage:
    1. Pull image mới nhất từ DockerHub.
    2. Deploy từng service (`customers`, `vets`, `visits`, `gateway`, `config`) bằng Helm chart.
    3. Kiểm tra trạng thái pod (`kubectl get pods`) → Running / Ready.

### ✅ Kết quả Stage
- Toàn bộ service được triển khai tự động lên Kubernetes.
- Jenkins đảm bảo quá trình deploy ổn định, có rollback khi gặp lỗi.

---

## 📊 Triển khai Monitoring Service trên Kubernetes bằng Helm

Để giám sát toàn hệ thống, **Prometheus + Grafana** được triển khai bằng **Helm chart**.

### ⚙️ Quy trình
1. Sử dụng Helm repository `prometheus-community` và `grafana`.
2. Prometheus thu thập metrics từ các service thông qua endpoint `/metrics`.
3. Grafana kết nối Prometheus, hiển thị dashboard CPU, memory, network, error rate, pod status.

### ✅ Kết quả Stage
Monitoring stack hoạt động trong namespace `monitoring`, giúp DevOps theo dõi hiệu năng và tình trạng hệ thống theo thời gian thực.

---

## 🪵 Triển khai Logging Stack (ELK) bằng Helm

Bên cạnh Monitoring, hệ thống cần **Logging stack** để tập trung log.

### 🧩 Thành phần
- **Elasticsearch**: lưu trữ log.
- **Logstash**: xử lý & gửi log.
- **Kibana**: giao diện xem log.

### ⚙️ Quy trình
1. Dùng Helm chart để triển khai toàn bộ **ELK stack** trong namespace `logging`.
2. Các service gửi log stdout/stderr đến **Logstash**, sau đó được đẩy vào **Elasticsearch**.
3. Kibana cung cấp UI trực quan để:
   - Theo dõi log theo service, namespace, container.
   - Lọc và tìm kiếm log theo thời gian.
   - Phát hiện lỗi và cảnh báo sớm.

### ✅ Kết quả Stage
Logging stack hoạt động ổn định trong namespace `logging`, giúp phân tích log tập trung và tăng khả năng giám sát hệ thống.

---


## 🔁 Flow CI/CD

Quy trình **CI/CD** đảm bảo việc phát triển, build, kiểm thử, đóng gói và triển khai ứng dụng được **tự động hóa hoàn toàn** từ khâu viết code đến giám sát hệ thống sau khi deploy.

---

![CI/CD Workflow](https://github.com/user-attachments/assets/6eebe43b-7b60-4a56-bce8-7036347cfb46)
*(Sơ đồ tổng quan quy trình CI/CD của hệ thống DevOps Petclinic)*

### 🧑‍💻 1. Developer (Local Stage)

- Developer thực hiện viết code và test cục bộ trên máy cá nhân.  
- Khi code ổn định, developer **push source code lên GitHub repository** (nhánh `main` hoặc `devops-final`).  
- Hành động này tự động **trigger pipeline CI** trong GitHub Actions.

---

### ⚙️ 2. Continuous Integration (CI) – GitHub Actions

**GitHub Actions** chịu trách nhiệm chạy pipeline CI khi có sự kiện `push` hoặc `pull request`.

Các bước chính trong pipeline:

1. Pull code mới nhất từ GitHub.  
2. Build từng service (sử dụng Maven/Gradle).  
3. Dockerize từng service → tạo Docker image.  
4. Push image lên **DockerHub** với tag version tương ứng (`v1.0`, `v1.1`, ...).  
5. Sau khi CI hoàn tất, GitHub Actions **gửi thông báo trạng thái build về Slack**.

---

### 🚀 3. Continuous Deployment (CD) – Jenkins

**Jenkins** nhận trigger từ CI hoặc được DevOps Engineer kích hoạt thủ công.

**Jenkins Pipeline (Jenkinsfile)** thực hiện:

- Pull Docker images mới nhất từ DockerHub.  
- Triển khai toàn bộ các service lên **Kubernetes Cluster** bằng **Helm chart**.  
- Kiểm tra trạng thái triển khai (pod status, replica health).  
- Sau khi deploy thành công, Jenkins gửi kết quả thông báo về **Slack Channel**.

---

### ☸️ 4. Kubernetes Deployment

Toàn bộ service từ `petclinic-app` được cài đặt vào **namespace riêng** trên Kubernetes (ví dụ: `petclinic-dev`).

Hệ thống đảm bảo:

- Mỗi service chạy trong **pod độc lập**.  
- Dùng **ConfigMap**, **Secret**, và **PersistentVolume** để quản lý cấu hình.  
- Có khả năng **rollback** khi deploy lỗi.  

---

### 📊 5. Monitoring & Logging

Sau khi hệ thống được deploy, hai stack hỗ trợ giám sát được kích hoạt:

#### 🔹 Monitoring Stack (Prometheus + Grafana)
- **Prometheus** thu thập metrics (CPU, RAM, Request count, Error rate) từ các pod.  
- **Grafana** kết nối đến Prometheus, hiển thị dashboard trực quan, giúp theo dõi hiệu năng theo thời gian thực.

#### 🔹 Logging Stack (ELK)
- **ELK Stack (Elasticsearch, Logstash, Kibana)** thu thập log từ container/pod trong cluster.  
- **Logstash** xử lý và gửi log đến **Elasticsearch**.  
- **Kibana** cho phép DevOps dễ dàng tìm kiếm, phân tích và theo dõi log của từng service.

---

### 💬 6. Notification

Mỗi bước trong quy trình CI/CD đều có cơ chế gửi **thông báo tự động về Slack**, bao gồm:

- CI hoàn tất (thành công / thất bại).  
- CD triển khai xong.  
- Cảnh báo lỗi nếu có service gặp sự cố.

---

### ✅ Tổng quan Flow CI/CD

| Giai đoạn | Công cụ sử dụng | Mục tiêu |
|------------|----------------|----------|
| **Local Dev** | VSCode, Git | Phát triển & test code cục bộ |
| **CI** | GitHub Actions, Docker | Build, test, dockerize, push image |
| **CD** | Jenkins, Helm, Kubernetes | Triển khai ứng dụng lên cluster |
| **Monitor** | Prometheus, Grafana | Theo dõi hiệu năng hệ thống |
| **Logging** | ELK Stack | Quản lý log tập trung |
| **Notification** | Slack | Báo cáo trạng thái pipeline |

---

