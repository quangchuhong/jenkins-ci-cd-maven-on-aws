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
Ý nghĩa:
- Stage 1: Build source thành file JAR (artifact) bằng Maven.
- Stage 2: Image runtime nhẹ hơn, chỉ chứa JAR đã build → giảm kích thước image & tăng bảo mật.
---
## 4. Jenkinsfile – CI pipeline
```bash
pipeline {
    agent any

    tools {
        maven 'Maven-3'
        jdk   'JDK-17'
    }

    environment {
        AWS_REGION      = "ap-southeast-1"
        AWS_ACCOUNT_ID  = "123456789012"
        ECR_REPO        = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/my-maven-app"

        SONARQUBE_SERVER   = "sonarqube"        // tên cấu hình SonarQube trong Jenkins
        SONAR_PROJECT_KEY  = "my-maven-app"

        APP_VERSION    = "1.0.0"
        GIT_SHORT      = "${env.GIT_COMMIT?.take(7)}"
    }

    stages {
        stage('Checkout from GitLab') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test (Maven)') {
            steps {
                sh 'mvn -B clean verify'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh """
                    mvn sonar:sonar \
                      -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                      -Dsonar.projectVersion=${APP_VERSION}-${GIT_SHORT}
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    timeout(time: 5, unit: 'MINUTES') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Quality Gate failed: ${qg.status}"
                        }
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.IMAGE_TAG = "${APP_VERSION}-${GIT_SHORT}"
                }
                sh """
                docker build -t ${ECR_REPO}:${IMAGE_TAG} .
                """
            }
        }

        stage('Trivy Scan Docker Image') {
            steps {
                sh """
                trivy image --exit-code 1 \
                  --severity CRITICAL,HIGH \
                  ${ECR_REPO}:${IMAGE_TAG}
                """
            }
        }

        stage('Push Image to ECR') {
            steps {
                withAWS(region: "${AWS_REGION}", credentials: 'aws-jenkins-creds') {
                    sh """
                    aws ecr get-login-password --region ${AWS_REGION} | \
                      docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                    docker push ${ECR_REPO}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Update Helm values (GitOps)') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'gitlab-gitops-creds',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_PASS'
                )]) {
                    sh """
                    # Clone chính repo này hoặc repo gitops riêng, ví dụ dùng chính app-repo:
                    git config --global user.email "ci@company.local"
                    git config --global user.name "jenkins-ci"

                    # Nếu dùng repo riêng:
                    # rm -rf gitops-repo
                    # git clone https://${GIT_USER}:${GIT_PASS}@gitlab.com/your-group/gitops-repo.git
                    # cd gitops-repo

                    cd helm/my-maven-app

                    # Cập nhật image.tag trong values.yaml
                    yq e '.image.tag = "${IMAGE_TAG}"' -i values.yaml

                    cd ../../
                    git add helm/my-maven-app/values.yaml
                    git commit -m "Update image tag to ${IMAGE_TAG}"
                    git push https://${GIT_USER}:${GIT_PASS}@gitlab.com/your-group/your-app-repo.git HEAD:main
                    """
                }
            }
        }
    }

    post {
        success {
            echo "CI pipeline OK – image ${ECR_REPO}:${IMAGE_TAG} ready & Helm values updated."
        }
        failure {
            echo "CI pipeline FAILED – kiểm tra log Jenkins stages."
        }
    }
}
```
## 5. Ý nghĩa chi tiết từng bước trong Workflow

### 5.1. GitLab → Jenkins (Trigger)

- **Ý nghĩa**: Khi dev push code/merge request, GitLab gửi webhook tới Jenkins để khởi chạy pipeline.
- Đảm bảo mọi thay đổi đều:
  - Được **build**,
  - Được **test**,
  - Được **scan** (chất lượng + bảo mật)
  trước khi có cơ hội được deploy.

---

### 5.2. Stage: `Build & Test (Maven)`

Lệnh chính:

```bash
mvn clean verify

clean: Xóa thư mục target cũ → đảm bảo build sạch.
verify: Chạy full Maven lifecycle:
   - compile → biên dịch mã nguồn.
   - test → chạy unit/integration tests.
   - package → đóng gói (JAR/WAR).
   - verify → thực thi các bước kiểm tra bổ sung (nếu cấu hình).
```
- **Ý nghĩa**:
   - Đảm bảo code biên dịch được và các test đều pass.
   - Nếu build hoặc test fail → dừng pipeline sớm, không tiếp tục sang bước scan, build image, deploy
---
### 5.3. Stage: SonarQube Analysis & Quality Gate
- Jenkin run:
  ```bash
  mvn sonar:sonar \
  -Dsonar.projectKey=... \
  -Dsonar.projectVersion=...
  ```
- **SonarQube thực hiện**:
   - Phân tích bugs, vulnerabilities, code smells.
   - Tính toán code coverage (nếu tích hợp JaCoCo).
   - Kiểm tra duplication, complexity, technical debt.

- Sau đó Jenkins chạy:
```bash
timeout(time: 5, unit: 'MINUTES') {
    def qg = waitForQualityGate()
    if (qg.status != 'OK') {
        error "Quality Gate failed: ${qg.status}"
    }
}
```
- **Ý nghĩa**:
   - waitForQualityGate đóng vai trò như một “cửa chặn” chất lượng:
   - Nếu Quality Gate không đạt (ví dụ: quá nhiều vulnerabilities, coverage thấp) → pipeline fail.
   - Giúp ngăn chặn code kém chất lượng lọt vào các bước sau (build image, deploy).
---
### 5.4. Stage: Build Docker Image
- Jenkins dùng Dockerfile multi-stage để:
   1. Build ứng dụng bằng Maven (tạo JAR),
   2. Đóng gói JAR vào image runtime nhẹ (JRE).
- Ví dụ lệnh:
```bash
docker build -t <ECR_REPO>:<IMAGE_TAG> .
```
- **Ý nghĩa**:
   - Chuẩn hóa môi trường chạy (runtime) dưới dạng container.
   - Đảm bảo artifact (JAR) đã test sẽ là thứ được deploy (không rebuild ở chỗ khác).
   - Tách biệt môi trường build (Maven + JDK full) và runtime (JRE nhẹ, ít bề mặt tấn công hơn).
---
### 5.5. Stage: Trivy Scan Docker Image
- Jenkins chạy:
```bash
trivy image --exit-code 1 \
  --severity CRITICAL,HIGH \
  <ECR_REPO>:<IMAGE_TAG>
```
- Trivy kiểm tra:
   - Lỗ hổng bảo mật (vulnerabilities) trong:
   - OS packages (Alpine, Debian, Ubuntu…),
   - Thư viện ngôn ngữ (Java dependencies, v.v.).a

- **Ý nghĩa**:
   - --exit-code 1 → nếu phát hiện vulnerabilities CRITICAL hoặc HIGH:
      - Stage fail,
      - Pipeline dừng.
   - Đảm bảo image an toàn trước khi được push lên registry & deploy.
   - Áp dụng nguyên tắc DevSecOps / shift-left security: bắt lỗi bảo mật càng sớm càng tốt.
---
### 5.6. Stage: Push Image to ECR
- Các bước chính:
   1. Đăng nhập vào AWS ECR bằng AWS CLI.
   2. Push image với tag duy nhất (version + commit hash).
Ví dụ:
```bash
aws ecr get-login-password --region <REGION> | \
  docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com

docker push <ECR_REPO>:<IMAGE_TAG>

```
- **Ý nghĩa**:
   - Lưu trữ image trong AWS ECR – registry riêng, an toàn, tích hợp tốt với EKS.
   - Mỗi build có một IMAGE_TAG duy nhất:
      - Dễ dàng trace từ container đang chạy → commit/source tương ứng.
      - Hỗ trợ rollback nhanh khi có sự cố.
---
### 5.7. Stage: Update Helm values (GitOps)
- Sử dụng yq chỉnh helm/my-maven-app/values.yaml:
   - image.tag = APP_VERSION-GIT_SHORT.
- Commit & push ngược về GitLab (hoặc repo GitOps riêng).
- **Ý nghĩa**:
   - Biến Git repo thành source of truth cho deployment.
   - Tách bạch CI (Jenkins) và CD (Argo CD):
      - Jenkins chỉ build, test, scan, cập nhật Git.
      - Argo CD tự deploy theo Git (GitOps).
---
### 6. Argo CD – CD lên EKS
continute...
