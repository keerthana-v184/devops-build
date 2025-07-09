# React E-Commerce Application Deployment using DevOps Practices

This project demonstrates the complete end-to-end deployment of a React-based E-Commerce application using industry-standard DevOps tools and workflows. The deployment pipeline includes Docker, GitHub, Jenkins, AWS EC2, DockerHub, and monitoring using Prometheus and Grafana.

---

## Table of Contents

* [Project Overview](#project-overview)
* [Workflow Overview](#workflow-overview)
* [1. Prerequisites](#1-prerequisites)
* [2. Repository Setup](#2-repository-setup)
* [3. Dockerization](#3-dockerization)

  * [Dockerfile](#dockerfile)
  * [docker-compose.yml](#docker-composeyml)
  * [Shell Scripts](#shell-scripts)
  * [.dockerignore](#dockerignore)
  * [.gitignore](#gitignore)
* [4. GitHub and Branching Strategy](#4-github-and-branching-strategy)
* [5. DockerHub Repositories](#5-dockerhub-repositories)
* [6. Jenkins Setup](#6-jenkins-setup)

  * [Install Jenkins](#install-jenkins)
  * [Jenkins Credentials](#jenkins-credentials)
  * [Configure Jenkins Pipeline](#configure-jenkins-pipeline)
  * [Jenkinsfile](#jenkinsfile)
* [7. AWS EC2 Deployment](#7-aws-ec2-deployment)
* [8. Monitoring with Prometheus & Grafana](#8-monitoring-with-prometheus--grafana)

  * [Monitoring Files](#monitoring-files)
* [9. Access and Verification](#9-access-and-verification)
* [10. Learning Outcome](#10-learning-outcome)

---

## Project Overview

This project focuses on deploying a React-based E-Commerce application using a complete DevOps pipeline. The goal is to achieve a fully automated, scalable, and production-grade deployment process using popular DevOps tools. The process includes:

* Dockerizing the React application.
* Using GitHub for version control and branch-based workflows.
* Creating a CI/CD pipeline with Jenkins that builds, pushes, and deploys Docker images based on the branch (dev or main).
* Hosting the application on an EC2 instance with proper security groups.
* Implementing open-source monitoring using Prometheus and Grafana to ensure high availability and health visibility.
* Sending email alerts when the application goes down.

This project demonstrates a real-world DevOps workflow from source to production.

---

## Workflow Overview

1. Fork and clone the repo to GitHub
2. Dockerize the app with `Dockerfile` and `docker-compose.yml`
3. Push to `dev` branch, build and push image to DockerHub (dev)
4. Merge to `main`, build and push to DockerHub (prod)
5. Jenkins CI/CD automates the process
6. EC2 instance (Node-React) runs the deployed container
7. Monitoring setup with Prometheus + Grafana

---

## 1. Prerequisites

* GitHub account and repo forked
* DockerHub account
* AWS account with 2 EC2 instances:

  * `Jenkins-Master`: Jenkins installed
  * `Node-React`: App deployed + monitoring
* Domain (optional) and email for alerts

---

## 2. Repository Setup

```bash
git clone https://github.com/keerthana-v184/devops-build.git
cd devops-build
git checkout -b dev
```

### File Structure

```
devops-build/
├── Dockerfile
├── docker-compose.yml
├── build.sh
├── deploy.sh
├── .dockerignore
├── .gitignore
├── Jenkinsfile
└── Monitoring/
    ├── docker-compose.yml
    ├── prometheus.yml
    ├── alert_rules.yml
    └── alertmanager.yml
```

---

## 3. Dockerization

### Dockerfile

```Dockerfile
FROM nginx:alpine
COPY build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### docker-compose.yml

```yaml
version: '3'
services:
  react-app:
    image: react-app
    build: .
    ports:
      - "80:80"
    restart: always
```

### Shell Scripts

#### build.sh

```bash
#!/bin/bash
docker build -t react-app .
```

#### deploy.sh

```bash
#!/bin/bash
docker-compose down
docker-compose up -d
```

```bash
chmod +x build.sh deploy.sh
```

### .dockerignore

```
node_modules
dist
.git
.gitignore
```

### .gitignore

```
node_modules
build
.env
.DS_Store
```

---

## 4. GitHub and Branching Strategy

```bash
git add .
git commit -m "Initial DevOps setup"
git push origin dev
```

* Push all files to `dev` branch
* Later merge `dev` to `main` for production

---

## 5. DockerHub Repositories

* Public: `iamkeerthana/dev`
* Private: `iamkeerthana/prod`

---

## 6. Jenkins Setup

### Jenkins Pipeline Creation Explained

1. **Create New Pipeline**:

   * Open Jenkins dashboard → Click on “New Item” → Enter name (e.g., `React-App-Deployment`) → Choose “Pipeline” → Click OK.

2. **Configure Source Code Management (SCM)**:

   * Select “Pipeline script from SCM”.
   * Choose Git, then provide your GitHub repo URL (e.g., `https://github.com/keerthana-v184/devops-build.git`).
   * Set Branches to build as `*/dev` and `*/main`.
   * Set Script Path to `Jenkinsfile` (make sure Jenkinsfile exists in repo).

3. **Credentials Setup**:

   * Use your GitHub personal access token as Git credentials.
   * DockerHub credentials for pushing images.
   * SSH key credentials to connect with the `Node-React` EC2 instance.

4. **Trigger Builds Automatically**:

   * Configure GitHub webhook (Settings → Webhooks → Payload URL: `http://<jenkins-ip>:8080/github-webhook/`).
   * Select `Just the push event`.

5. **Save and Run**:

   * Click Save.
   * Perform push/merge to trigger pipeline.

### Jenkinsfile

```groovy
pipeline {
    agent { label 'node-react-agent' }

    environment {
        DOCKER_REGISTRY = "docker.io"
        DEV_IMAGE = "iamkeerthana/dev:dev"
        PROD_IMAGE = "iamkeerthana/prod:prod"
        DOCKER_CREDS = "docker-hub-credentials"
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Setup Image Tag') {
            steps {
                script {
                    def branch = env.GIT_BRANCH?.replace('origin/', '') ?: sh(
                        script: "git rev-parse --abbrev-ref HEAD",
                        returnStdout: true
                    ).trim()

                    if (branch == 'dev') {
                        env.DOCKER_IMAGE = "${DOCKER_REGISTRY}/${DEV_IMAGE}"
                    } else if (branch == 'main') {
                        env.DOCKER_IMAGE = "${DOCKER_REGISTRY}/${PROD_IMAGE}"
                    } else {
                        error("Unsupported branch: ${branch}")
                    }

                    echo "Using Docker Image: ${env.DOCKER_IMAGE}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${env.DOCKER_IMAGE} ."
                sh "docker image inspect ${env.DOCKER_IMAGE} > /dev/null"
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: env.DOCKER_CREDS,
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}
                    '''
                }
            }
        }

        stage('Deploy App on Node') {
            steps {
                sh '''
                    docker pull ${DOCKER_IMAGE}
                    docker stop react-app || true
                    docker rm react-app || true
                    docker run -d --name react-app -p 80:80 ${DOCKER_IMAGE}
                '''
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
            sh "docker rmi ${env.DOCKER_IMAGE} || true"
            cleanWs()
        }
        success {
            echo "Successfully deployed ${env.DOCKER_IMAGE} on node-react-agent"
        }
        failure {
            echo "Deployment failed for ${env.DOCKER_IMAGE}"
        }
    }
}
```

---

## 7. AWS EC2 Deployment

* `Jenkins-Master`: Jenkins setup, public subnet
* `Node-React`: Docker + Prometheus + Grafana + app container

---

## 8. Monitoring with Prometheus & Grafana

### Monitoring Setup Explained

1. **Objective**:

   * Monitor the health of the deployed React application.
   * Alert via email if the app is down.

2. **Tools Used**:

   * **Prometheus**: Metrics collection and monitoring.
   * **Grafana**: Visualization and dashboards.
   * **Node Exporter**: Host-level metrics.
   * **Blackbox Exporter**: URL-based health probe.
   * **AlertManager**: Sends alert notifications (email).

3. **Implementation**:

   * All services are defined in a `docker-compose.yml` file.
   * `blackbox-exporter` probes the React app URL to check availability.
   * `prometheus.yml` is configured to scrape metrics from node-exporter, blackbox-exporter, and itself.
   * `alert_rules.yml` defines the rule to fire an alert if the React app is unreachable.
   * `alertmanager.yml` defines SMTP settings for sending alert emails.

4. **Visualization**:

   * Grafana is accessible on port `3000`.
   * Dashboard imported using ID `7587`, linked with Prometheus.
   * Shows live status of the app, node stats, and service availability.

5. **Testing Alerts**:

   * Stopping the container triggers an alert.
   * Email is sent instantly to the configured address.

### docker-compose.yml

```yaml
version: '3'
services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus:/etc/prometheus
    ports:
      - "9090:9090"

  alertmanager:
    image: prom/alertmanager
    volumes:
      - ./alertmanager:/etc/alertmanager
    ports:
      - "9093:9093"

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"

  node-exporter:
    image: prom/node-exporter
    ports:
      - "9100:9100"

  blackbox-exporter:
    image: prom/blackbox-exporter
    ports:
      - "9115:9115"
```

### prometheus.yml

```yaml
global:
  scrape_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - "alert_rules.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - http://35.173.187.48
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
      - source_labels: []
        target_label: instance
        replacement: React-App
```

### alert\_rules.yml

```yaml
groups:
  - name: blackbox-alerts
    rules:
      - alert: ReactAppDown
        expr: probe_success == 0
        for: 15s
        labels:
          severity: critical
        annotations:
          summary: "React App is Down"
          description: "The React app at http://35.173.187.48 is unreachable."
```

### alertmanager.yml

```yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: '<your-email@gmail.com>'
  smtp_auth_username: '<your-email@gmail.com>'
  smtp_auth_password: '<your-app-password>'

route:
  receiver: 'email-alert'

receivers:
  - name: 'email-alert'
    email_configs:
      - to: '<your-email@gmail.com>'
        send_resolved: true
```

---

## 9. Access and Verification

* App URL: [http://35.173.187.48](http://35.173.187.48)
* Grafana: [http://35.173.187.48:3000](http://35.173.187.48:3000)
* DockerHub:

  * [dev](https://hub.docker.com/repository/docker/iamkeerthana/dev)
  * [prod](https://hub.docker.com/repository/docker/iamkeerthana/prod)

---

## 10. Learning Outcome

* Dockerized and deployed a production-ready React app
* Built a CI/CD pipeline with Jenkins, GitHub, and DockerHub
* Automated deployment using shell scripts and webhooks
* Monitored the app using Prometheus + Grafana with alert notifications
* Understood full lifecycle of DevOps deployment from code to production
