# VProfile — End-to-End CI/CD Pipeline with Jenkins, Docker, Kubernetes & Helm

Production-grade CI/CD pipeline that automates building, testing, code-quality analysis, containerization, and deployment of a multi-tier Java web application onto a Kubernetes cluster using Helm charts.

![CI/CD](https://img.shields.io/badge/CI%2FCD-Jenkins-D24939?style=flat&logo=jenkins&logoColor=white)
![Docker](https://img.shields.io/badge/Container-Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Orchestration-Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/Packaging-Helm-0F1689?style=flat&logo=helm&logoColor=white)
![SonarQube](https://img.shields.io/badge/Code%20Quality-SonarQube-4E9BCD?style=flat&logo=sonarqube&logoColor=white)
![Slack](https://img.shields.io/badge/Notifications-Slack-4A154B?style=flat&logo=slack&logoColor=white)

---

## CI/CD Pipeline — Stage Breakdown

The entire pipeline is defined as code in the [`Jenkinsfile`](Jenkinsfile) with **9 automated stages**:

| # | Stage | What It Does |
|---|-------|-------------|
| 1 | **Build** | Compiles the Java source and packages it into `vprofile-v2.war` using Maven |
| 2 | **Unit Tests** | Runs the full test suite via `mvn test` |
| 3 | **Checkstyle Analysis** | Static code analysis for coding standard violations |
| 4 | **SonarQube Analysis** | Pushes code metrics, test reports, coverage, and checkstyle results to SonarQube server |
| 5 | **Quality Gate** | Waits for SonarQube quality gate verdict — pipeline **aborts** if gate fails |
| 6 | **Docker Build** | Builds the container image, tags it with `v$BUILD_ID` and `latest` |
| 7 | **DockerHub Login** | Authenticates to DockerHub using Jenkins credentials |
| 8 | **Push Image** | Pushes all tags to DockerHub, then cleans up local images |
| 9 | **Kubernetes Deploy** | Deploys/upgrades the entire stack to K8s via `helm upgrade --install` on a KOPS agent node |

**Post-build:** Sends a Slack notification to the `#jenkinscicd` channel with build status, version, and a link to the build.

---

## Containerization

The application is containerized via a multi-step approach defined in the [`Dockerfile`](Dockerfile):

```dockerfile
FROM tomcat:9-jre11
RUN rm -rf /usr/local/tomcat/webapps/*
COPY target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

- **Base image:** Tomcat 9 on JRE 11
- **Artifact:** The Maven-built WAR is deployed as `ROOT.war` so the app is served at `/`
- **Port:** 8080

---

## Kubernetes Deployment

The project contains **multiple Kubernetes manifest strategies** showcasing different approaches:

### Approach 1 — Raw Manifests (`kubernetes/vpro-app/`)

Production-ready manifests with:

| Resource | File | Details |
|----------|------|---------|
| **Secret** | `app-secret.yml` | Stores DB and RMQ passwords (base64 encoded) |
| **MySQL Deployment** | `vprodbdep.yml` | 1 replica, pulls password from `app-secret` via `secretKeyRef` |
| **Memcached Deployment** | `mcdep.yml` | 1 replica, port 11211 |
| **RabbitMQ Deployment** | `rmq-dep.yml` | 1 replica, port 15672, credentials from `app-secret` |
| **App Deployment** | `vproappdep.yml` | 1 replica with **initContainers** that wait for DB & cache DNS resolution before starting |
| **Services** | `db-CIP.yml`, `mc-CIP.yml`, `rmq-CIP-service.yml` | All backend services exposed as **ClusterIP** (internal only) |
| **App Service** | `vproapp-service.yml` | Exposed as **LoadBalancer** on port 80 |

**Key pattern — initContainers:** The app pod uses two `busybox` init containers that perform DNS lookups (`nslookup`) for `vprodb` and `vprocache01` services, ensuring the app only starts once its dependencies are ready.

### Approach 2 — Combined Manifest with HPA (`kubernetes/app.yml`)

A single-file deployment with:
- **3 replicas** with CPU resource limits/requests
- **HorizontalPodAutoscaler** — scales between 1–10 pods at 50% CPU utilization
- **LoadBalancer** service

### Approach 3 — Individual Pod/Service Manifests (`kubernetes/db/`, `kubernetes/memcache/`, `kubernetes/tomapp/`)

Standalone Pod and Deployment manifests for each tier, useful for learning and testing individual components with NodePort services.

---

## Helm Chart

The [`helm/vprofilecharts/`](helm/vprofilecharts/) directory packages the entire stack as a **Helm chart** for repeatable, parameterized deployments:

```
helm/vprofilecharts/
├── Chart.yaml          # Chart metadata (v0.1.0, application type)
├── values.yaml         # Default values (overridden by Jenkins)
└── templates/
    ├── app-secret.yml       # K8s Secret for DB & RMQ passwords
    ├── vproappdep.yml       # App Deployment (image from {{ .Values.appImage }})
    ├── vproapp-service.yml  # App Service (LoadBalancer)
    ├── vprodbdep.yml        # MySQL Deployment (secret-based credentials)
    ├── db-CIP.yml           # MySQL ClusterIP Service
    ├── mcdep.yml            # Memcached Deployment
    ├── mc-CIP.yml           # Memcached ClusterIP Service
    ├── rmq-dep.yml          # RabbitMQ Deployment (secret-based credentials)
    └── rmq-CIP-service.yml  # RabbitMQ ClusterIP Service
```

**How Jenkins deploys it:**
```bash
helm upgrade --install --force vprofile-stack helm/vprofilecharts \
  --set appImage=rajatrulaniya/app:latest \
  --namespace prod
```

The `appImage` value is injected at deploy time, so the chart always picks up the freshly built Docker image from the pipeline.

---

## Infrastructure & Services

| Component | Image | Purpose | Exposed Port |
|-----------|-------|---------|-------------|
| **VProfile App** | `rajatrulaniya/app` (custom) | Java web app on Tomcat | 8080 → 80 (LB) |
| **MySQL** | `vprofile/vprofiledb` | Relational database (`accounts` DB) | 3306 (ClusterIP) |
| **Memcached** | `memcached` (official) | User data caching with active/standby failover | 11211 (ClusterIP) |
| **RabbitMQ** | `rabbitmq` (official) | Async message broker (fanout exchange) | 5672 (ClusterIP) |

---

## Tools & Technologies

| Category | Tools |
|----------|-------|
| **CI/CD** | Jenkins (Declarative Pipeline) |
| **Containerization** | Docker |
| **Container Registry** | DockerHub |
| **Orchestration** | Kubernetes (KOPS on AWS) |
| **Package Management** | Helm 3 |
| **Code Quality** | SonarQube, Checkstyle, JaCoCo |
| **Build Tool** | Maven 3 |
| **Notifications** | Slack |
| **Version Control** | Git / GitHub |

---

## Jenkins Prerequisites

To run this pipeline, the Jenkins server needs:

| Requirement | Details |
|-------------|---------|
| **JDK** | OracleJDK 17 (configured as `OracleJDK17` in Global Tool Config) |
| **Maven** | Maven 3+ (configured as `maven` in Global Tool Config) |
| **SonarQube Scanner** | Configured as `sonarscanner` |
| **SonarQube Server** | Configured as `sonarserver` with webhook for Quality Gate |
| **Docker** | Installed on Jenkins agent |
| **Helm** | Installed on KOPS agent node |
| **Credentials** | `dockerhub` (Username/Password) in Jenkins Credentials |
| **Slack Plugin** | Configured with workspace integration for `#jenkinscicd` |
| **KOPS Agent** | A Jenkins agent labeled `KOPS` with `kubectl` and `helm` access to the cluster |

---

## Application Quick Reference

The underlying application is a **Spring MVC** user profile management system. Only relevant if you need to build locally:

| Item | Value |
|------|-------|
| **Language** | Java 8 |
| **Framework** | Spring MVC 4.2, Spring Security, Spring Data JPA |
| **Database** | MySQL 5.6+ (schema: `src/main/resources/accountsdb.sql`) |
| **Build** | `mvn clean install` → produces `target/vprofile-v2.war` |
| **Run Locally** | Deploy WAR to Tomcat 9, or use `mvn jetty:run` |
| **DB Setup** | `mysql -u <user> -p accounts < src/main/resources/accountsdb.sql` |

---

## Project Structure

```
├── Dockerfile                  # Container image definition
├── Jenkinsfile                 # Declarative CI/CD pipeline (9 stages)
├── pom.xml                     # Maven build config
├── helm/
│   └── vprofilecharts/         # Helm chart for full-stack K8s deployment
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/          # K8s manifests with Helm templating
├── kubernetes/
│   ├── app.yml                 # Combined deployment + HPA
│   ├── vpro-app/               # Production manifests (Secrets, initContainers)
│   ├── db/                     # Standalone DB manifests
│   ├── memcache/               # Standalone Memcached manifests
│   └── tomapp/                 # Standalone App manifests
└── src/                        # Java application source code
```

---

## How It All Comes Together

1. Developer pushes code to **GitHub**
2. **Jenkins** picks up the change and runs the pipeline
3. Code is **built**, **tested**, and **analyzed** (Checkstyle + SonarQube)
4. Pipeline halts if **Quality Gate** fails — no bad code reaches production
5. A **Docker image** is built and pushed to **DockerHub** with a unique build version tag
6. **Helm** deploys/upgrades the full stack (app + MySQL + Memcached + RabbitMQ) to a **Kubernetes cluster** on AWS (via KOPS)
7. The app pod waits for database and cache readiness via **initContainers** before starting
8. **Slack** notification is sent with build result and version info

---
