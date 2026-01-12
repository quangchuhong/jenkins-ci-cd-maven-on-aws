# CI/CD Workflow cho Ứng Dụng Maven trên EKS

## 1. Kiến trúc tổng quan

Các thành phần:

- **GitLab**: Lưu source code, Dockerfile, Helm chart, Jenkinsfile.
- **Jenkins**: Thực thi pipeline CI (build, test, scan, build image, push ECR, update GitOps).
- **Maven**: Build & test ứng dụng Java (Spring Boot hoặc Java app).
- **SonarQube**: Phân tích chất lượng code (bug, code smell, coverage).
- **Trivy**: Scan image Docker để tìm vulnerabilities.
- **Docker + AWS ECR**: Build và lưu trữ container image.
- **Helm Chart**: Đóng gói manifest K8s.
- **Argo CD**: GitOps – tự động deploy version mới lên EKS khi Helm values thay đổi.
- **Amazon EKS**: Môi trường chạy app.

Luồng chính:

1. Dev push code lên **GitLab**.
2. GitLab webhook → trigger **Jenkins** pipeline.
3. Jenkins:
   - Build & test Maven.
   - Phân tích chất lượng với SonarQube.
   - Build image Docker, scan bằng Trivy.
   - Push image lên AWS ECR.
   - Cập nhật repo GitOps (Helm values) → commit/tag image mới.
4. **Argo CD** theo dõi repo GitOps → sync → deploy lên **EKS**.

---

## 2. Cấu trúc repo (app-repo)

Ví dụ cấu trúc:

```bash
app-repo/
├── src/
├── pom.xml
├── Dockerfile
├── Jenkinsfile
└── helm/
    └── my-maven-app/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── deployment.yaml
            ├── service.yaml
            └── ingress.yaml (nếu dùng)
```
---
## 3. Dockerfile (multi-stage build cho Maven)

Stage 1: Build với Maven
```
FROM maven:3.9.9-eclipse-temurin-17 AS builder
WORKDIR /build
COPY pom.xml .
COPY src ./src
RUN mvn -B clean package -DskipTests
```
Stage 2: Runtime image
```
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=builder /build/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

