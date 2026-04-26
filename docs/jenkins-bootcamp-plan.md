# Jenkins Coaching Playbook (VN) — Foundation to Production

Tài liệu này tổng hợp kiến thức Jenkins theo hướng thực chiến CI/CD cho senior developer:
- Bắt đầu từ nền tảng (khái niệm, cài đặt, quản trị).
- Đi đến Jenkins Pipeline production-like (Jenkinsfile, credentials, quality gates, security).
- Hoàn thiện mini-project end-to-end với Node.js + Docker (+ tùy chọn Kubernetes/AWS).

---

## 1) 5 câu hỏi khảo sát đầu vào (Intake)

1. Mức độ hiện tại với Jenkins/CI/CD (0–10), đã từng viết Jenkinsfile chưa?
2. Stack chính đang dùng là Node.js hay Java Spring Boot (kèm version)?
3. Muốn chạy Jenkins local bằng Docker Compose hay trên server/cloud ngay từ đầu?
4. Có đang dùng Docker/Kubernetes/AWS không, mức độ tới đâu (POC/prod)?
5. Mục tiêu thực tế trong 4–6 tuần: học nền tảng, build pipeline production, hay áp dụng thẳng vào repo hiện tại?

---

## 2) Jenkins Fundamentals — Kiến thức cốt lõi

### 2.1 Jenkins là gì
- Jenkins là automation server phục vụ CI/CD.
- Jenkins orchestration workflow từ checkout -> build -> test -> package -> deploy -> notify.
- Jenkins không thay thế công cụ build; Jenkins điều phối toolchain (npm, Maven, Docker, kubectl...).

### 2.2 Kiến trúc
- **Controller**: quản lý job, queue, UI, credentials, plugin.
- **Agent**: máy/worker chạy build thực tế.
- **Executor**: slot thực thi trên agent.
- **Workspace**: thư mục làm việc theo job/build.

### 2.3 Job types
- Freestyle (legacy, nhanh cho demo).
- Pipeline (khuyến nghị production).
- Multibranch Pipeline (chuẩn cho Git workflow).
- Organization Folder (quản lý nhiều repo/team).

### 2.4 Pipeline styles
- **Declarative**: rõ cấu trúc, dễ maintain, khuyến nghị team-scale.
- **Scripted**: linh hoạt cao (Groovy thuần), khó chuẩn hóa hơn.

### 2.5 Jenkinsfile lifecycle
- Checkout source.
- Parse Jenkinsfile.
- Allocate agent.
- Chạy stages + steps.
- Post actions (archive/report/notify/cleanup).

---

## 3) Setup Jenkins Local với Docker Compose (khuyến nghị cho học + POC)

### 3.1 File cần có
- `docker-compose.yml`
- Thư mục volume `jenkins_home/`
- (Tùy chọn) `plugins.txt` hoặc JCasC config

### 3.2 Docker Compose mẫu
```yaml
version: '3.8'
services:
  jenkins:
    image: jenkins/jenkins:lts-jdk17
    container_name: jenkins
    user: root
    ports:
      - '8080:8080'
      - '50000:50000'
    volumes:
      - ./jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
```

### 3.3 Bootstrap commands
```bash
mkdir -p jenkins_home
docker compose up -d
docker ps
docker logs -f jenkins
```

### 3.4 Lỗi thường gặp
- Port 8080 conflict.
- Permission denied với `jenkins_home`.
- Docker socket không mount -> fail stage Docker build.
- Plugin tải lỗi do proxy/DNS.

---

## 4) Jenkinsfile Declarative — Cấu trúc chuẩn production

### 4.1 Jenkinsfile mẫu full pipeline (10 stage)
```groovy
pipeline {
  agent any

  options {
    timestamps()
    timeout(time: 30, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '20'))
    disableConcurrentBuilds()
  }

  parameters {
    choice(name: 'DEPLOY_ENV', choices: ['dev', 'staging'], description: 'Target environment')
    booleanParam(name: 'SKIP_DOCKER_PUSH', defaultValue: false, description: 'Skip image push')
  }

  environment {
    APP_NAME = 'nodejs-demo'
    IMAGE_REPO = 'your-dockerhub-user/nodejs-demo'
    IMAGE_TAG = "${env.BUILD_NUMBER}"
    DOCKER_IMAGE = "${IMAGE_REPO}:${IMAGE_TAG}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Install dependencies') {
      steps {
        sh 'npm ci'
      }
    }

    stage('Lint') {
      steps {
        sh 'npm run lint'
      }
    }

    stage('Unit test') {
      steps {
        sh 'npm test -- --ci --reporters=default --reporters=jest-junit'
      }
      post {
        always {
          junit testResults: 'junit.xml', allowEmptyResults: true
        }
      }
    }

    stage('Build') {
      steps {
        sh 'npm run build'
      }
      post {
        success {
          archiveArtifacts artifacts: 'dist/**', fingerprint: true
        }
      }
    }

    stage('Docker build') {
      steps {
        sh 'docker build -t ${DOCKER_IMAGE} .'
      }
    }

    stage('Security scan basic') {
      steps {
        sh 'trivy image --exit-code 0 --severity HIGH,CRITICAL ${DOCKER_IMAGE} || true'
      }
    }

    stage('Push image') {
      when {
        expression { return !params.SKIP_DOCKER_PUSH }
      }
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
          sh 'docker push ${DOCKER_IMAGE}'
        }
      }
    }

    stage('Deploy mock') {
      steps {
        sh 'echo "Deploy ${DOCKER_IMAGE} to ${DEPLOY_ENV} (mock)"'
      }
    }
  }

  post {
    always {
      echo 'Pipeline finished'
      cleanWs()
    }
    success {
      echo 'Build success'
    }
    failure {
      echo 'Build failed'
    }
  }
}
```

### 4.2 Giải thích từng block
- `pipeline`: root block cho Declarative Pipeline.
- `agent`: nơi chạy pipeline (`any`, label, docker agent...).
- `options`: hardening cơ bản (`timeout`, `timestamps`, `buildDiscarder`).
- `parameters`: cho phép rerun build linh hoạt theo môi trường/use-case.
- `environment`: biến chung, không chứa plain-text secrets.
- `stages`: chia luồng CI/CD thành checkpoint rõ ràng.
- `steps`: lệnh thực thi trong từng stage.
- `post`: xử lý sau build (archive/report/notify/cleanup).
- `withCredentials`: inject secrets an toàn ở runtime.

### 4.3 Declarative vs Scripted (ngắn gọn)
- Chọn **Declarative** khi cần chuẩn hóa, readability, maintainability.
- Chọn **Scripted** khi flow phức tạp cần logic Groovy động cao.

---

## 5) Credentials & Secret Management

### 5.1 Loại credentials thường dùng
- Username/Password
- Secret text (token)
- Secret file (kubeconfig, key file)
- SSH private key

### 5.2 Best practices
- Không hard-code secret trong Jenkinsfile/repo.
- Dùng `withCredentials` scope nhỏ nhất có thể.
- Không `echo` secret ra log.
- Tách credential theo môi trường (dev/staging/prod).
- Rotate credential định kỳ.

---

## 6) Plugin & Tooling nên có cho production baseline

- Pipeline / Pipeline: Stage View
- Git / GitHub Branch Source
- Credentials Binding
- Docker Pipeline
- JUnit
- Warnings NG
- Workspace Cleanup
- Timestamper
- Build Timeout
- Blue Ocean (optional)
- OWASP Dependency-Check hoặc Trivy integration (tuỳ stack)

> Lưu ý: Càng ít plugin càng tốt; chỉ cài plugin thật sự cần để giảm attack surface.

---

## 7) Observability, Debugging & Troubleshooting

### 7.1 Cách debug chuẩn
1. Đọc Console Output theo stage.
2. Xác minh workspace (`pwd`, `ls -la`, artifacts).
3. Kiểm tra agent/tool versions (`node -v`, `npm -v`, `docker -v`).
4. Kiểm tra credentials binding scope.
5. Re-run stage/job với input tối thiểu.

### 7.2 10 lỗi phổ biến
1. `Jenkinsfile not found` (sai path/branch).
2. `npm ci` fail do lockfile lệch.
3. Unit test treo do process không thoát.
4. Docker build fail vì thiếu daemon/socket.
5. Push image fail do sai credential/permission.
6. `junit.xml` không đúng path.
7. Disk đầy do không clean workspace/artifacts.
8. Build timeout không cấu hình.
9. Race condition khi concurrent builds.
10. Plugin conflict sau update.

---

## 8) Security baseline cho Jenkins

- Bật Matrix-based security hoặc RBAC (nếu dùng CloudBees/enterprise).
- Bật CSRF protection.
- Disable anonymous read.
- Dùng HTTPS/reverse proxy.
- Backup `jenkins_home` định kỳ.
- Giới hạn quyền credentials theo folder/job.
- Pin version plugin/core, update có kiểm soát.

---

## 9) Scaling patterns

- Single controller + static agents (nhỏ).
- Ephemeral agents (Docker/Kubernetes) để clean và scale.
- Shared Library để tái sử dụng pipeline logic.
- Multibranch + branch protections + PR checks.

---

## 10) Mini-project end-to-end (khung thực hành)

### 10.1 Kiến trúc dự án đề xuất
- App: Node.js REST API đơn giản.
- CI: Jenkins Declarative Pipeline.
- Container: Docker image.
- Registry: Docker Hub/ECR.
- Deploy: mock bằng Docker Compose hoặc thật lên Kubernetes/AWS.

### 10.2 Stage bắt buộc trong pipeline
1. Checkout
2. Install dependencies
3. Lint
4. Unit test
5. Build
6. Docker build
7. Security scan cơ bản
8. Push image
9. Deploy mock/deploy thật
10. Post actions (artifacts, reports, notifications)

### 10.3 Tiêu chí production-like
- Secret không hard-code.
- Build reproducible (`npm ci`, fixed base image).
- Artifact và image tag có version rõ.
- Stage tách bạch, fail fast.
- Có timeout/timestamps/build discarder.
- Có scan và policy xử lý findings.
- Logs đủ để debug.

---

## 11) Lộ trình 10 buổi chi tiết (teaching plan)

### Buổi 1 — Jenkins foundation + local setup
- **Mục tiêu:** Hiểu core architecture và tự dựng Jenkins local.
- **Lab:** Docker Compose + unlock + tạo pipeline đầu tiên.
- **Kiểm tra:** Phân biệt controller/agent/executor/workspace.
- **Checklist:** Jenkins up, login, run build pass.

### Buổi 2 — Declarative pipeline fundamentals
- **Mục tiêu:** Nắm `pipeline/agent/environment/stages/steps/post/parameters`.
- **Lab:** Viết Jenkinsfile skeleton có 3 stage.
- **Kiểm tra:** So sánh Declarative vs Scripted.
- **Checklist:** Pipeline readable, có post actions.

### Buổi 3 — CI stages cho Node.js
- **Mục tiêu:** Checkout/install/lint/test/build chuẩn.
- **Lab:** Chạy full CI stage 1→5.
- **Kiểm tra:** Vì sao `npm ci` phù hợp CI.
- **Checklist:** Test report hiển thị Jenkins.

### Buổi 4 — Artifacts, reports, quality gate
- **Mục tiêu:** Lưu artifacts + test reports + fail policy.
- **Lab:** `archiveArtifacts`, `junit`.
- **Kiểm tra:** Artifact vs image.
- **Checklist:** Report truy vết theo build.

### Buổi 5 — Dockerization & image strategy
- **Mục tiêu:** Build image, tag strategy, reproducibility.
- **Lab:** Stage Docker build + metadata.
- **Kiểm tra:** Rủi ro khi chỉ dùng tag `latest`.
- **Checklist:** Image build thành công từ pipeline.

### Buổi 6 — Credentials & hardening
- **Mục tiêu:** Quản lý secret an toàn.
- **Lab:** Docker registry login bằng credentials binding.
- **Kiểm tra:** Vì sao không echo secrets.
- **Checklist:** Không lộ secret trong log.

### Buổi 7 — Security scan cơ bản
- **Mục tiêu:** Scan dependency/image và hành vi fail/pass.
- **Lab:** Trivy hoặc Dependency-Check.
- **Kiểm tra:** Chọn threshold severity.
- **Checklist:** Có report scan + quyết định rõ.

### Buổi 8 — Push image + deploy mock
- **Mục tiêu:** Tự động push và deploy mock.
- **Lab:** Push image rồi deploy target.
- **Kiểm tra:** Immutable artifact là gì.
- **Checklist:** Deploy dùng đúng image tag.

### Buổi 9 — Production hardening & ops
- **Mục tiêu:** Tăng ổn định: timeout/retry/parallel/notify.
- **Lab:** Harden Jenkinsfile.
- **Kiểm tra:** Khi nào cần disable concurrent builds.
- **Checklist:** Pipeline stable và dễ vận hành.

### Buổi 10 — Capstone + roadmap Kubernetes/AWS
- **Mục tiêu:** Demo full end-to-end + review.
- **Lab:** Chạy toàn bộ pipeline từ commit tới deploy.
- **Kiểm tra:** Chiến lược rollback khả thi.
- **Checklist:** Ready áp dụng cho repo thực tế.

---

## 12) Bài tập tự luyện sau mỗi buổi (template)

- **Deliverables:** Jenkinsfile + log build + ảnh stage view.
- **Tự đánh giá:** Build time, fail rate, mức reproducibility.
- **Nâng cao:** Tách shared library, thêm manual approval cho production.

---

## 13) Checklist triển khai Jenkins cho team (quick audit)

- [ ] Jenkins core và plugins được pin/update định kỳ.
- [ ] RBAC/permission theo principle of least privilege.
- [ ] Credentials không hard-code, có rotation plan.
- [ ] Pipelines có timeout/timestamps/discarder.
- [ ] CI có lint/test/build + report rõ ràng.
- [ ] Docker image có versioning và provenance.
- [ ] Security scan có policy pass/fail.
- [ ] Có backup/restore cho Jenkins state.
- [ ] Có dashboard/notifications theo team channel.

---

## 14) Assumptions & Risk

### Assumptions
- Team dùng Git-based workflow (PR/branch).
- Có Docker runtime ở môi trường Jenkins hoặc agents.
- Có registry để push image.

### Risk
- Quá nhiều plugin gây xung đột/khó maintain.
- Pipeline chạy trên agent không chuẩn hóa tool versions.
- Secret scope quá rộng gây rủi ro bảo mật.
- Không cleanup workspace dẫn tới cạn disk.

