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
  
**Repository:** [petclinic-deploy](https://github.com/petclinic-devops/petclinic-deploy)
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











# Distributed version of the Spring PetClinic Sample Application built with Spring Cloud and Spring AI

[![Build Status](https://github.com/spring-petclinic/spring-petclinic-microservices/actions/workflows/maven-build.yml/badge.svg)](https://github.com/spring-petclinic/spring-petclinic-microservices/actions/workflows/maven-build.yml)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

This microservices branch was initially derived from [AngularJS version](https://github.com/spring-petclinic/spring-petclinic-angular1) to demonstrate how to split sample Spring application into [microservices](http://www.martinfowler.com/articles/microservices.html).
To achieve that goal, we use Spring Cloud Gateway, Spring Cloud Circuit Breaker, Spring Cloud Config, Micrometer Tracing, Resilience4j, Open Telemetry 
and the Eureka Service Discovery from the [Spring Cloud Netflix](https://github.com/spring-cloud/spring-cloud-netflix) technology stack.

[![Open in GitHub Codespaces](https://github.com/codespaces/badge.svg)](https://codespaces.new/spring-petclinic/spring-petclinic-microservices)

[![Open in Codeanywhere](https://codeanywhere.com/img/open-in-codeanywhere-btn.svg)](https://app.codeanywhere.com/#https://github.com/spring-petclinic/spring-petclinic-microservices)

## Starting services locally without Docker

Every microservice is a Spring Boot application and can be started locally using IDE or `../mvnw spring-boot:run` command.
Please note that supporting services (Config and Discovery Server) must be started before any other application (Customers, Vets, Visits and API).
Startup of Tracing server, Admin server, Grafana and Prometheus is optional.
If everything goes well, you can access the following services at given location:
* Discovery Server - http://localhost:8761
* Config Server - http://localhost:8888
* AngularJS frontend (API Gateway) - http://localhost:8080
* Customers, Vets, Visits and GenAI Services - random port, check Eureka Dashboard 
* Tracing Server (Zipkin) - http://localhost:9411/zipkin/ (we use [openzipkin](https://github.com/openzipkin/zipkin/tree/main/zipkin-server))
* Admin Server (Spring Boot Admin) - http://localhost:9090
* Grafana Dashboards - http://localhost:3000
* Prometheus - http://localhost:9091

You can tell Config Server to use your local Git repository by using `native` Spring profile and setting
`GIT_REPO` environment variable, for example:
`-Dspring.profiles.active=native -DGIT_REPO=/projects/spring-petclinic-microservices-config`

## Starting services locally with docker-compose
In order to start entire infrastructure using Docker, you have to build images by executing
``bash
./mvnw clean install -P buildDocker
``
This requires `Docker` or `Docker desktop` to be installed and running.

Alternatively you can also build all the images on `Podman`, which requires Podman or Podman Desktop to be installed and running.
```bash
./mvnw clean install -PbuildDocker -Dcontainer.executable=podman
```
By default, the Docker OCI image is build for an `linux/amd64` platform.
For other architectures, you could change it by using the `-Dcontainer.platform` maven command line argument.
For instance, if you target container images for an Apple M2, you could use the command line with the `linux/arm64` architecture:
```bash
./mvnw clean install -P buildDocker -Dcontainer.platform="linux/arm64"
```

Once images are ready, you can start them with a single command
`docker compose up` or `podman-compose up`. 

Containers startup order is coordinated with the `service_healthy` condition of the Docker Compose [depends-on](https://github.com/compose-spec/compose-spec/blob/main/spec.md#depends_on) expression 
and the [healthcheck](https://github.com/compose-spec/compose-spec/blob/main/spec.md#healthcheck) of the service containers. 
After starting services, it takes a while for API Gateway to be in sync with service registry,
so don't be scared of initial Spring Cloud Gateway timeouts. You can track services availability using Eureka dashboard
available by default at http://localhost:8761.

The `main` branch uses an Eclipse Temurin with Java 17 as Docker base image.

*NOTE: Under MacOSX or Windows, make sure that the Docker VM has enough memory to run the microservices. The default settings
are usually not enough and make the `docker-compose up` painfully slow.*


## Starting services locally with docker-compose and Java
If you experience issues with running the system via docker-compose you can try running the `./scripts/run_all.sh` script that will start the infrastructure services via docker-compose and all the Java based applications via standard `nohup java -jar ...` command. The logs will be available under `${ROOT}/target/nameoftheapp.log`. 

Each of the java based applications is started with the `chaos-monkey` profile in order to interact with Spring Boot Chaos Monkey. You can check out the (README)[scripts/chaos/README.md] for more information about how to use the `./scripts/chaos/call_chaos.sh` helper script to enable assaults.

## Understanding the Spring Petclinic application

[See the presentation of the Spring Petclinic Framework version](http://fr.slideshare.net/AntoineRey/spring-framework-petclinic-sample-application)

[A blog post introducing the Spring Petclinic Microsevices](http://javaetmoi.com/2018/10/architecture-microservices-avec-spring-cloud/) (french language)

You can then access petclinic here: http://localhost:8080/

## Microservices Overview

This project consists of several microservices:
- **Customers Service**: Manages customer data.
- **Vets Service**: Handles information about veterinarians.
- **Visits Service**: Manages pet visit records.
- **GenAI Service**: Provides a chatbot interface to the application.
- **API Gateway**: Routes client requests to the appropriate services.
- **Config Server**: Centralized configuration management for all services.
- **Discovery Server**: Eureka-based service registry.

Each service has its own specific role and communicates via REST APIs.


![Spring Petclinic Microservices screenshot](docs/application-screenshot.png)


**Architecture diagram of the Spring Petclinic Microservices**

![Spring Petclinic Microservices architecture](docs/microservices-architecture-diagram.jpg)

## Integrating the Spring AI Chatbot

Spring Petclinic integrates a Chatbot that allows you to interact with the application in a natural language. Here are some examples of what you could ask:

1. Please list the owners that come to the clinic.
2. Are there any vets that specialize in surgery?
3. Is there an owner named Betty?
4. Which owners have dogs?
5. Add a dog for Betty. Its name is Moopsie.
6. Create a new owner.

![Screenshot of the chat dialog](docs/spring-ai.png)

This `spring-petlinic-genai-service` microservice currently supports **OpenAI** (default) or **Azure's OpenAI** as the LLM provider.
In order to start the microservice, perform the following steps:

1. Decide which provider you want to use. By default, the `spring-ai-openai-spring-boot-starter` dependency is enabled. 
   You can change it to `spring-ai-azure-openai-spring-boot-starter`in the `pom.xml`.
2. Create an OpenAI API key or a Azure OpenAI resource in your Azure Portal.
   Refer to the [OpenAI's quickstart](https://platform.openai.com/docs/quickstart) or [Azure's documentation](https://learn.microsoft.com/en-us/azure/ai-services/openai/) for further information on how to obtain these.
   You only need to populate the provider you're using - either openai, or azure-openai.
   If you don't have your own OpenAI API key, don't worry!
   You can temporarily use the `demo` key, which OpenAI provides free of charge for demonstration purposes.
   This `demo` key has a quota, is limited to the `gpt-4o-mini` model, and is intended solely for demonstration use.
   With your own OpenAI account, you can test the `gpt-4o` model by modifying the `deployment-name` property of the `application.yml` file.
3. Export your API keys and endpoint as environment variables:
    * either OpenAI:
    ```bash
    export OPENAI_API_KEY="your_api_key_here"
    ```
    * or Azure OpenAI:
    ```bash
    export AZURE_OPENAI_ENDPOINT="https://your_resource.openai.azure.com"
    export AZURE_OPENAI_KEY="your_api_key_here"
    ```

## In case you find a bug/suggested improvement for Spring Petclinic Microservices

Our issue tracker is available here: https://github.com/spring-petclinic/spring-petclinic-microservices/issues

## Database configuration

In its default configuration, Petclinic uses an in-memory database (HSQLDB) which gets populated at startup with data.
A similar setup is provided for MySql in case a persistent database configuration is needed.
Dependency for Connector/J, the MySQL JDBC driver is already included in the `pom.xml` files.

### Start a MySql database

You may start a MySql database with docker:

```
docker run -e MYSQL_ROOT_PASSWORD=petclinic -e MYSQL_DATABASE=petclinic -p 3306:3306 mysql:8.4.5
```
or download and install the MySQL database (e.g., MySQL Community Server 8.4.5 LTS), which can be found here: https://dev.mysql.com/downloads/

### Use the Spring 'mysql' profile

To use a MySQL database, you have to start 3 microservices (`visits-service`, `customers-service` and `vets-services`)
with the `mysql` Spring profile. Add the `--spring.profiles.active=mysql` as program argument.

By default, at startup, database schema will be created and data will be populated.
You may also manually create the PetClinic database and data by executing the `"db/mysql/{schema,data}.sql"` scripts of each 3 microservices. 
In the `application.yml` of the [Configuration repository], set the `initialization-mode` to `never`.

If you are running the microservices with Docker, you have to add the `mysql` profile into the (Dockerfile)[docker/Dockerfile]:
```
ENV SPRING_PROFILES_ACTIVE docker,mysql
```
In the `mysql section` of the `application.yml` from the [Configuration repository], you have to change 
the host and port of your MySQL JDBC connection string. 

## Custom metrics monitoring

Grafana and Prometheus are included in the `docker-compose.yml` configuration, and the public facing applications
have been instrumented with [MicroMeter](https://micrometer.io) to collect JVM and custom business metrics.

A JMeter load testing script is available to stress the application and generate metrics: [petclinic_test_plan.jmx](spring-petclinic-api-gateway/src/test/jmeter/petclinic_test_plan.jmx)

![Grafana metrics dashboard](docs/grafana-custom-metrics-dashboard.png)

### Using Prometheus

* Prometheus can be accessed from your local machine at http://localhost:9091

### Using Grafana with Prometheus

* An anonymous access and a Prometheus datasource are setup.
* A `Spring Petclinic Metrics` Dashboard is available at the URL http://localhost:3000/d/69JXeR0iw/spring-petclinic-metrics.
You will find the JSON configuration file here: [docker/grafana/dashboards/grafana-petclinic-dashboard.json]().
* You may create your own dashboard or import the [Micrometer/SpringBoot dashboard](https://grafana.com/dashboards/4701) via the Import Dashboard menu item.
The id for this dashboard is `4701`.

### Custom metrics
Spring Boot registers a lot number of core metrics: JVM, CPU, Tomcat, Logback... 
The Spring Boot auto-configuration enables the instrumentation of requests handled by Spring MVC.
All those three REST controllers `OwnerResource`, `PetResource` and `VisitResource` have been instrumented by the `@Timed` Micrometer annotation at class level.

* `customers-service` application has the following custom metrics enabled:
  * @Timed: `petclinic.owner`
  * @Timed: `petclinic.pet`
* `visits-service` application has the following custom metrics enabled:
  * @Timed: `petclinic.visit`

## Looking for something in particular?

| Spring Cloud components         | Resources  |
|---------------------------------|------------|
| Configuration server            | [Config server properties](spring-petclinic-config-server/src/main/resources/application.yml) and [Configuration repository] |
| Service Discovery               | [Eureka server](spring-petclinic-discovery-server) and [Service discovery client](spring-petclinic-vets-service/src/main/java/org/springframework/samples/petclinic/vets/VetsServiceApplication.java) |
| API Gateway                     | [Spring Cloud Gateway starter](spring-petclinic-api-gateway/pom.xml) and [Routing configuration](/spring-petclinic-api-gateway/src/main/resources/application.yml) |
| Docker Compose                  | [Spring Boot with Docker guide](https://spring.io/guides/gs/spring-boot-docker/) and [docker-compose file](docker-compose.yml) |
| Circuit Breaker                 | [Resilience4j fallback method](spring-petclinic-api-gateway/src/main/java/org/springframework/samples/petclinic/api/boundary/web/ApiGatewayController.java)  |
| Grafana / Prometheus Monitoring | [Micrometer implementation](https://micrometer.io/), [Spring Boot Actuator Production Ready Metrics] |

|  Front-end module | Files |
|-------------------|-------|
| Node and NPM      | [The frontend-maven-plugin plugin downloads/installs Node and NPM locally then runs Bower and Gulp](spring-petclinic-ui/pom.xml)  |
| Bower             | [JavaScript libraries are defined by the manifest file bower.json](spring-petclinic-ui/bower.json)  |
| Gulp              | [Tasks automated by Gulp: minify CSS and JS, generate CSS from LESS, copy other static resources](spring-petclinic-ui/gulpfile.js)  |
| Angular JS        | [app.js, controllers and templates](spring-petclinic-ui/src/scripts/)  |

## Pushing to a Docker registry

Docker images for `linux/amd64` and `linux/arm64` platforms have been published into DockerHub 
in the [springcommunity](https://hub.docker.com/u/springcommunity) organization.
You can pull an image:
```bash
docker pull springcommunity/spring-petclinic-config-server
```
You may prefer to build then push images to your own Docker registry.

### Choose your Docker registry

You need to define your target Docker registry.
Make sure you're already logged in by running `docker login <endpoint>` or `docker login` if you're just targeting Docker hub.

Setup the `REPOSITORY_PREFIX` env variable to target your Docker registry.
If you're targeting Docker hub, simple provide your username, for example:
```bash
export REPOSITORY_PREFIX=springcommunity
```

For other Docker registries, provide the full URL to your repository, for example:
```bash
export REPOSITORY_PREFIX=harbor.myregistry.com/petclinic
```

To push Docker image for the `linux/amd64` and the `linux/arm64` platform to your own registry, please use the command line:
```bash
mvn clean install -Dmaven.test.skip -P buildDocker -Ddocker.image.prefix=${REPOSITORY_PREFIX} -Dcontainer.build.extraarg="--push" -Dcontainer.platform="linux/amd64,linux/arm64"
```

The `scripts/pushImages.sh` and `scripts/tagImages.sh` shell scripts could also be used once you build your image with the `buildDocker` maven profile.
The `scripts/tagImages.sh` requires to declare the `VERSION` env variable.

## Compiling the CSS

There is a `petclinic.css` in `spring-petclinic-api-gateway/src/main/resources/static/css`.
It was generated from the `petclinic.scss` source, combined with the [Bootstrap](https://getbootstrap.com/) library.
If you make changes to the `scss`, or upgrade Bootstrap, you will need to re-compile the CSS resources
using the Maven profile `css` of the `spring-petclinic-api-gateway`module.
```bash
cd spring-petclinic-api-gateway
mvn generate-resources -P css
```

## Interesting Spring Petclinic forks

The Spring Petclinic `main` branch in the main [spring-projects](https://github.com/spring-projects/spring-petclinic)
GitHub org is the "canonical" implementation, currently based on Spring Boot and Thymeleaf.

This [spring-petclinic-microservices](https://github.com/spring-petclinic/spring-petclinic-microservices/) project is one of the [several forks](https://spring-petclinic.github.io/docs/forks.html) 
hosted in a special GitHub org: [spring-petclinic](https://github.com/spring-petclinic).
If you have a special interest in a different technology stack
that could be used to implement the Pet Clinic then please join the community there.


## Contributing

The [issue tracker](https://github.com/spring-petclinic/spring-petclinic-microservices/issues) is the preferred channel for bug reports, features requests and submitting pull requests.

For pull requests, editor preferences are available in the [editor config](.editorconfig) for easy use in common text editors. Read more and download plugins at <http://editorconfig.org>.


[Configuration repository]: https://github.com/spring-petclinic/spring-petclinic-microservices-config
[Spring Boot Actuator Production Ready Metrics]: https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-metrics.html

## Supported by

[![JetBrains logo](https://resources.jetbrains.com/storage/products/company/brand/logos/jetbrains.svg)](https://jb.gg/OpenSourceSupport)
