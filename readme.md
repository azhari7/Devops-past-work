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

## 🧪 SonarQube Integration for GitLab CI/CD

SonarQube is used for static code analysis, integrated with GitLab CI/CD pipelines via Docker Compose.

## 🌐 HAProxy as SSL-Terminating Reverse Proxy

Implemented in production to serve multiple domains with SSL offloading and domain-based routing, replacing NGINX.

## Updated: CI/CD Migration to AWS

Migrates CI/CD to use Docker Hub instead of Amazon ECR, updating Terraform and buildspec configurations.

---

This comprehensive setup ensures a secure, scalable, and efficient deployment of the mobile API service with integrated monitoring and CI/CD practices.
