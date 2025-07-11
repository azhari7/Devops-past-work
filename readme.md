# 📦 Mobile API Deployment for Government Service

This project is a Go-based mobile backend API containerized with a multi-stage Docker build and deployed to a Kubernetes cluster using Helm. It includes advanced features such as Vault for secrets injection, HPA for autoscaling, and is exposed via NGINX Ingress with TLS.

## 📚 Table of Contents
- [Dockerfile Overview](#-dockerfile-overview)
- [Kubernetes Deployment with Helm](#-kubernetes-deployment-with-helm)
- [Vault Integration](#-vault-integration)
- [Horizontal Pod Autoscaler (HPA)](#-horizontal-pod-autoscaler-hpa)
- [Ingress (NGINX)](#-ingress-nginx)
- [Local Debugging with Delve](#-local-debugging-with-delve)
- [Security Best Practices](#-security-best-practices)
- [Helm Integration](#-helm-integration)
- [CI/CD Pipeline](#-cicd-pipeline)
- [Monitoring Stack for Laravel Apps](#-monitoring-stack-for-laravel-apps)
- [SonarQube Integration](#-sonarqube-integration)
- [HAProxy as SSL-Terminating Reverse Proxy](#-haproxy-as-ssl-terminating-reverse-proxy)
- [CI/CD Migration to AWS](#-cicd-migration-to-aws)

## 🧱 Dockerfile Overview: Multi-Stage Go Build

**Why multi-stage?**
To keep the final image slim and production-safe while compiling with debug symbols in a separate builder stage.

### Key Stages:
- **Stage 1**: Build with debug symbols
  ```dockerfile
  FROM golang:1.24 AS builder
  ...
  ```
- **Stage 2**: Lightweight Alpine runtime
  ```dockerfile
  FROM alpine:latest
  ...
  ```

**Key Features:**
- Uses `go build -gcflags="all=-N -l"` to allow Delve debugging.
- Adds a non-root user for secure container practices.
- App binary placed in `/usr/local/bin/mobile-api`.
- Runtime environment includes `.env` and `captcha.ttf` injected via Helm.

## 🚀 Kubernetes Deployment with Helm

### 🛠️ Deployment.yaml
Purpose: Deploy the `mobile-api` app as a Pod managed by a Deployment controller.

**Cool Bits:**
- Secrets injected via HashiCorp Vault Agent Sidecar.
- Dynamic Vault token retrieval:
  ```bash
  export VAULT_TOKEN=$(cat /vault/secrets/token)
  . /app/.env
  exec mobile-api
  ```
- Resources controlled via `values.yaml`.
- GitOps-ready with Helm templating.

## 🔐 Vault Integration

Configurations inject a short-lived Vault token into the pod at runtime, ensuring tight security without baking secrets into images.

## 🔁 Horizontal Pod Autoscaler (HPA)

Monitors CPU and Memory Utilization to dynamically scale between `minReplicas` and `maxReplicas`, allowing for automatic handling of traffic spikes.

## 🌐 Ingress (NGINX)

**Highlights:**
- TLS-enabled and configurable via Helm values.
- Increased body size limit for file uploads.

## 🧪 Local Debugging with Delve

Because the binary is compiled with `-gcflags="all=-N -l"`, you can attach Delve for real-time debugging in development environments.

## 🛡️ Security Best Practices

- Runs as a non-root user (`appuser`).
- Secrets managed with Vault.
- Minimal final image (Alpine).
- Uses `imagePullSecrets` for private registries.

## 🧩 Helm Integration

All deployment configurations are abstracted to `values.yaml`, simplifying deployment across different environments.

## ⚙️ CI/CD Pipeline: Build & Deploy Mobile API

**Tooling:** GitHub Actions + Docker + Helm + Harbor Registry + Kubernetes

### Environments:
- **dev** → Development
- **staging** → Staging
- **main** → Production

### 📋 Job: build_and_deploy
- **Setup Helm**: Installs Helm v3.9.3.
- **Inject `values.yaml`**: Decodes base64-encoded values for Helm.
- **Login to Helm & Docker Registry**: Secure login using stored secrets.
- **Build & Push Docker Image**: Uses GitHub credentials to pull private dependencies.
- **Deploy to Kubernetes**: Runs `helm upgrade --install` for deployments.

## 📊 Monitoring Stack for Laravel Apps

A Docker Compose-based monitoring stack for Laravel applications implementing Prometheus, Grafana, Loki, and Promtail.
🔧 Stack Overview

This Docker Compose-based monitoring stack was deployed to provide full observability for application metrics and logs:
Component	Purpose	Port
Prometheus	Metrics collection and scraping	9090
Grafana	Dashboard visualization and alerting	3001
Loki	Centralized log storage	3100
Promtail	Log shipper from Laravel apps	(N/A)
🛠 Tech Stack

    App: Laravel (PHP)

    Container Management: Docker Compose

    Monitoring: Prometheus + Grafana

    Log Aggregation: Loki + Promtail

    Architecture: Local + Bridge network (monitoring_monitoring)

📁 Directory Layout
```
monitoring/
├── docker-compose.yml            # Prometheus + Grafana
├── prometheus/
│   └── prometheus.yaml           # Prometheus config
└── logs/
    ├── docker-compose.yml        # Loki + Promtail
    ├── loki-config.yaml          # Loki config
    └── promtail-config.yaml      # Promtail scrape rules
```
🚀 Prometheus + Grafana (docker-compose)
```
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus/prometheus.yaml:/etc/prometheus/prometheus.yaml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin_1@m
      - GF_USERS_ALLOW_SIGN_UP=false
    ports:
      - "3001:3000"
    depends_on:
      - prometheus

volumes:
  prometheus_data:
  grafana_data:

networks:
  monitoring:
    driver: bridge
```
Notes:

    Prometheus uses hot-reload enabled (--web.enable-lifecycle)

    Grafana port exposed on 3001 to avoid conflict with other services

    Admin password set via environment (to be rotated in production)

📝 Loki + Promtail for Laravel Logs
```
services:
  loki:
    image: grafana/loki:latest
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml
    ports:
      - "3100:3100"

  promtail:
    image: grafana/promtail:latest
    volumes:
      - ./promtail-config.yaml:/etc/promtail/config.yaml
      - /var/www:/var/www  # Laravel logs
```
Promtail monitors Laravel logs in /var/www/storage/logs/*.log and ships them to Loki.
📜 Sample Promtail Config
```
server:
  http_listen_port: 9080
  grpc_listen_port: 0

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: laravel_logs
    static_configs:
      - targets: [localhost]
        labels:
          job: laravel
          __path__: /var/www/storage/logs/*.log
```
📊 Grafana Dashboards

Custom dashboards were created to monitor:

    App performance (Laravel metrics, request time, status codes)

    Host metrics (CPU, memory via Node Exporter)

    Log trends (Laravel errors, warnings via Loki)

    Integrated alerting via Grafana alert rules and optionally email or Slack

## 🧪 SonarQube Integration for GitLab CI/CD

SonarQube is used for static code analysis, integrated with GitLab CI/CD pipelines via Docker Compose.

📦 SonarQube + PostgreSQL via Docker Compose

```
version: "3"

services:
  sonarqube:
    image: sonarqube:community
    container_name: sonarqube
    depends_on:
      - db
    ports:
      - "9000:9000"
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonarqube
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
    networks:
      - sonarnet

  db:
    image: postgres:13
    container_name: sonardb
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar
      POSTGRES_DB: sonarqube
    volumes:
      - postgresql:/var/lib/postgresql/data
    networks:
      - sonarnet

volumes:
  sonarqube_data:
  sonarqube_extensions:
  postgresql:

networks:
  sonarnet:
    driver: bridge
```
🧪 CI/CD Integration (GitLab sonar-scanner Stage)

In .gitlab-ci.yml, a dedicated scan stage was added:
```
stages:
  - build
  - test
  - sonar

sonarqube_scan:
  stage: sonar
  image:
    name: sonarsource/sonar-scanner-cli:latest
    entrypoint: [""]
  script:
    - sonar-scanner \
        -Dsonar.projectKey=my-project \
        -Dsonar.sources=. \
        -Dsonar.host.url=http://sonarqube:9000 \
        -Dsonar.login=$SONAR_TOKEN
  only:
    - merge_requests
    - main
    - dev
```
🔐 Secrets

Stored in GitLab CI/CD variables:

    SONAR_TOKEN: SonarQube user token with analysis permissions

🖼 SonarQube UI

    Available at: http://localhost:9000

    Initial user: admin / admin (change ASAP)

    Quality gates applied to enforce standards for merge/push pipelines

    Reports:

        Code coverage

        Complexity

        Duplications

        Bugs / vulnerabilities

🧠 Key Learnings (Approximate)
Contribution	Tools Used
Deployed static analysis platform	Docker Compose, SonarQube
Integrated quality scan into CI/CD	GitLab Pipelines + Sonar Scanner
Managed secrets + tokens securely	GitLab CI/CD Variables
Helped enforce code quality gates	SonarQube UI, Quality Profiles


## 🌐 HAProxy as SSL-Terminating Reverse Proxy

Implemented in production to serve multiple domains with SSL offloading and domain-based routing, replacing NGINX.
🛠 Use Case

The system replaces NGINX by using HAProxy as the reverse proxy layer for:

    SSL termination (TLS v1.2+)

    Host-based routing to backend web apps (Laravel or static sites)

    Centralized logging + socket-based admin control

    Clean URL redirection to HTTPS

⚙️ Key Features
Feature	Description
SSL Multi-Cert Handling	HAProxy uses crt directive to serve multiple domains
Host-based Routing	Uses ACLs to map domains/subdomains to backends
Security	TLSv1.2 minimum, hardened ciphers (Mozilla intermediate)
Performance	Keep-alive settings + connection timeout tuning
Admin Socket	HAProxy UNIX socket for external automation or monitoring
Forward Headers	Injects X-Forwarded-Proto and client IP

🧱 Sample HAProxy Structure (Censored & Cleaned)

frontend https_front
    bind *:80
    bind *:443 ssl crt /etc/haproxy/certs/example1.com.pem crt /etc/haproxy/certs/example2.com.pem
    mode http
    http-request redirect scheme https unless { ssl_fc }
    option forwardfor
    http-request set-header X-Forwarded-Proto https

    # Routing by host
    acl is_example1 hdr(host) -i example1.com
    use_backend backend_example1 if is_example1

    acl is_sub_example1 hdr(host) -m end .example1.com
    use_backend backend_example1 if is_sub_example1

    acl is_example2 hdr(host) -i example2.org
    use_backend backend_example2 if is_example2

    acl is_sub_example2 hdr(host) -m end .example2.org
    use_backend backend_example2 if is_sub_example2

🖥 Backend Example

backend backend_example1
    server example1_server 10.1.2.3:80 check

backend backend_example2
    server example2_server 10.1.2.4:80 check

    🧠 IP addresses and domains have been masked to protect client privacy.

🔐 SSL & Security Config

ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets
ssl-default-bind-ciphers ECDHE-RSA-AES256-GCM-SHA384:...

    Using TLSv1.2+ only

    Mozilla-recommended intermediate cipher suites

    No session tickets = better forward secrecy

🧪 Extra Config (Global & Defaults)

global
    log /dev/log local0
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    daemon

defaults
    mode http
    timeout client 300s
    timeout server 300s
    option forwardfor
    option http-server-close



## Updated: CI/CD Migration to AWS

Migrates CI/CD to use Docker Hub instead of Amazon ECR, updating Terraform and buildspec configurations.

CI/CD Migration to AWS (Docker Hub Edition)

    ⚠️ Note: This is an approximate project  — done as a prototype, not in official employment.

🔄 Registry Switch

Instead of using Amazon ECR, the Docker image is:

    Tagged like: docker.io/yourusername/rails-app:latest

    Pushed to your Docker Hub account

    Pullable by Kubernetes in EKS

🔐 Docker Hub Secrets

You'll need to store these in AWS Systems Manager (SSM) or GitHub Secrets:

    DOCKER_USERNAME

    DOCKER_PASSWORD

🔁 Updated Terraform + buildspec.yml
```
buildspec.yml (CI/CD logic)

version: 0.2

env:
  secrets-manager:
    DOCKER_USERNAME: "dockerhub-username"
    DOCKER_PASSWORD: "dockerhub-password"

phases:
  pre_build:
    commands:
      - echo Logging into Docker Hub...
      - echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
  build:
    commands:
      - echo Building image...
      - docker build -t $DOCKER_USERNAME/rails-app:latest .
  post_build:
    commands:
      - echo Pushing image to Docker Hub...
      - docker push $DOCKER_USERNAME/rails-app:latest

artifacts:
  files:
    - imagedigest.txt
```
🔧 Terraform Snippets
```
Remove ECR section. Keep CodeBuild and CodePipeline mostly the same.

Environment variables for Docker Hub creds:

environment {
  compute_type = "BUILD_GENERAL1_SMALL"
  image        = "aws/codebuild/standard:5.0"
  type         = "LINUX_CONTAINER"
  environment_variables = [
    {
      name  = "DOCKER_USERNAME"
      value = "your-dockerhub-user"
      type  = "PLAINTEXT" # Or "SECRETS_MANAGER"
    },
    {
      name  = "DOCKER_PASSWORD"
      value = "your-dockerhub-pass"
      type  = "PLAINTEXT"
    }
  ]
  privileged_mode = true # needed for Docker
}
```
    For better security, store those in AWS Secrets Manager and reference them via type = "SECRETS_MANAGER".

🚀 Deployment

1. Code pushed to GitHub
2. CodePipeline triggers CodeBuild
3. CodeBuild builds Docker image
4. Image pushed to Docker Hub
5. A Helm release or K8s deployment watches/pulls latest image

---

This comprehensive setup ensures a secure, scalable, and efficient deployment of the mobile API service with integrated monitoring and CI/CD practices.
