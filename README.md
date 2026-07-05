<div align="center">

# 🚀 Namegen DevSecOps Platform

### Secure CI/CD delivery of a Node.js and MongoDB application on Amazon EKS

[![AWS](https://img.shields.io/badge/AWS-EKS%20%7C%20ECR%20%7C%20EBS-FF9900?logo=amazonaws&logoColor=white)](https://aws.amazon.com/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-EKS-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-CI%2FCD-2088FF?logo=githubactions&logoColor=white)](https://github.com/features/actions)
[![CodeQL](https://img.shields.io/badge/CodeQL-SAST-181717?logo=github&logoColor=white)](https://codeql.github.com/)
[![Trivy](https://img.shields.io/badge/Trivy-Image_Scanning-1904DA?logo=aqua&logoColor=white)](https://trivy.dev/)
[![MongoDB](https://img.shields.io/badge/MongoDB-StatefulSet-47A248?logo=mongodb&logoColor=white)](https://www.mongodb.com/)
[![Node.js](https://img.shields.io/badge/Node.js-Application-339933?logo=nodedotjs&logoColor=white)](https://nodejs.org/)

**Push code. Scan automatically. Build once. Deploy declaratively. Persist safely.**

</div>

---

## 📌 Overview

**Namagen** is a cloud-native Random Name Generator and Saver deployed on **Amazon Elastic Kubernetes Service (EKS)** in `us-west-2`.

The project demonstrates an end-to-end **DevSecOps delivery pipeline** in which GitHub Actions scans the code and container image, builds and publishes the image to Amazon ECR, and applies Kubernetes manifests to the EKS control plane. At runtime, an AWS Network Load Balancer exposes the Node.js application while MongoDB remains isolated behind an internal Kubernetes service and stores its data on an Amazon EBS-backed persistent volume.

### What this project demonstrates

- Automated CI/CD from `git push` to EKS deployment
- Static application analysis with **CodeQL**
- Container vulnerability scanning with **Trivy**
- Private image distribution through **Amazon ECR**
- Declarative Kubernetes deployments using `kubectl apply`
- Public application ingress through an **AWS Network Load Balancer**
- Internal-only MongoDB communication through a Kubernetes service
- Stateful database operation with a **StatefulSet**, PVC, and Amazon EBS
- MongoDB authentication with credentials injected at runtime
- Clear separation between build-time automation and runtime infrastructure

---

## 🏗️ Architecture

![Namagen DevSecOps and Amazon EKS architecture](docs/architecture.png)

### Delivery path

```text
Developer
   │
   └── git push
          │
          ▼
GitHub Repository
          │
          ▼
GitHub Actions
   ├── CodeQL scan
   ├── Trivy scan
   ├── Docker build
   ├── Push image to Amazon ECR
   └── Apply Kubernetes manifests to Amazon EKS
```

### Runtime request path

```text
User Browser
    │
    ▼
AWS Network Load Balancer :80
    │
    ▼
Namagen Node.js Pod :8080
    │
    ▼
Internal MongoDB Service :27017
    │
    ▼
MongoDB StatefulSet
    │
    ▼
PersistentVolumeClaim → Amazon EBS
```

---

## 🧱 Runtime Topology

| Layer | Component | Responsibility |
|---|---|---|
| Edge | AWS Network Load Balancer | Accepts public TCP traffic on port `80` and routes it to the application service |
| Application | `namagen-app-deployment` | Runs the Node.js application on container port `8080` |
| Service discovery | Kubernetes Service | Provides stable routing between the application and MongoDB |
| Database | `mongodb-statefulset` | Runs MongoDB on port `27017` with authentication enabled |
| Persistence | PVC + Amazon EBS CSI Driver | Preserves MongoDB data independently of Pod lifecycle |
| Registry | Amazon ECR | Stores private, versioned container images |
| Orchestration | Amazon EKS | Schedules workloads and manages the Kubernetes control plane |

### Deployment profile represented in the diagram

| Setting | Value |
|---|---|
| AWS Region | `us-west-2` |
| Application runtime | Node.js |
| Application container port | `8080` |
| Application replicas | `1` |
| Database | MongoDB `3.6` |
| Database container port | `27017` |
| Database replicas | `1` |
| Public listener | TCP `80` |
| Persistent storage | Amazon EBS through a Kubernetes PVC |

---

## 🔄 CI/CD Pipeline

A push to the deployment branch triggers the GitHub Actions workflow.

### 1. Build and security validation

The pipeline performs security checks before deployment:

- **CodeQL** inspects the source code for security weaknesses.
- **Trivy** scans the container image and its operating-system packages for known vulnerabilities.
- Pipeline scripts fail fast when a required build or security step does not complete successfully.

### 2. Container build

Docker creates an immutable application image. The recommended tagging strategy is:

```text
<account-id>.dkr.ecr.us-west-2.amazonaws.com/<repository>:<git-commit-sha>
```

Using the Git commit SHA provides a direct link between the deployed container and the source revision that produced it.

### 3. Push to Amazon ECR

The validated image is pushed to a private Amazon ECR repository. EKS then pulls that exact image during the rollout.

### 4. Deploy to Amazon EKS

The workflow applies the Kubernetes manifests and updates the application image:

```bash
kubectl apply -f k8s/
kubectl set image deployment/namagen-app-deployment \
  namagen-app="$ECR_REGISTRY/$ECR_REPOSITORY:$GITHUB_SHA"
kubectl rollout status deployment/namagen-app-deployment --timeout=180s
kubectl rollout status statefulset/mongodb-statefulset --timeout=180s
```

---

## 🔐 Security Model

### Passwordless AWS authentication

GitHub Actions should authenticate to AWS through **OpenID Connect (OIDC)** and an assumable IAM role. This avoids storing long-lived AWS access keys in GitHub.

Recommended GitHub repository variables:

| Variable | Example |
|---|---|
| `AWS_REGION` | `us-west-2` |
| `EKS_CLUSTER_NAME` | `namagen-eks` |
| `ECR_REPOSITORY` | `namagen` |
| `AWS_ROLE_ARN` | `arn:aws:iam::<account-id>:role/github-actions-namagen` |

### Database isolation

MongoDB is not exposed to the public internet. The application reaches it through an internal Kubernetes service on port `27017`.

### Secret handling

Do not commit a connection string containing a real username or password. Store MongoDB credentials in a Kubernetes Secret and construct `MONGODB_URL` from `secretKeyRef` values in the Deployment manifest.

Example one-time secret creation:

```bash
kubectl create namespace namagen --dry-run=client -o yaml | kubectl apply -f -

kubectl -n namagen create secret generic mongodb-credentials \
  --from-literal=username='genuser' \
  --from-literal=password='<replace-with-a-strong-password>' \
  --dry-run=client -o yaml | kubectl apply -f -
```

Recommended logical connection format:

```text
mongodb://<username>:<password>@mongodb-service:27017/namagen?authSource=namagen
```

For production environments, consider sourcing credentials from AWS Secrets Manager through an external-secrets integration rather than creating them manually.

---

## 📂 Suggested Repository Structure

```text
.
├── .github/
│   └── workflows/
│       └── deploy.yml
├── app/
│   ├── package.json
│   └── src/
├── docker/
│   └── mongodb/
│       ├── Dockerfile
│       └── init-mongo.js
├── k8s/
│   ├── namespace.yaml
│   ├── app-deployment.yaml
│   ├── app-service.yaml
│   ├── mongodb-service.yaml
│   ├── mongodb-statefulset.yaml
│   └── storage-class.yaml
├── scripts/
│   └── deploy.sh
├── docs/
│   └── architecture.png
├── Dockerfile
├── .dockerignore
└── README.md
```

---

## ✅ Prerequisites

Before deploying, install and configure:

- An AWS account with permission to use EKS, ECR, IAM, EC2, Elastic Load Balancing, and EBS
- AWS CLI
- `kubectl`
- Docker
- Git
- An existing Amazon EKS cluster in `us-west-2`
- Amazon EBS CSI Driver installed in the cluster
- A private Amazon ECR repository
- A GitHub OIDC provider and IAM role for GitHub Actions

Confirm the local tools:

```bash
aws --version
kubectl version --client
docker --version
git --version
```

---

## 🚀 Deployment

### 1. Clone the repository

```bash
git clone https://github.com/<your-github-user>/devops-final-project.git
cd devops-final-project
```

### 2. Configure access to the EKS cluster

```bash
aws eks update-kubeconfig \
  --region us-west-2 \
  --name <eks-cluster-name>
```

Verify connectivity:

```bash
kubectl cluster-info
kubectl get nodes
```

### 3. Create the MongoDB credentials

```bash
kubectl create namespace namagen --dry-run=client -o yaml | kubectl apply -f -

kubectl -n namagen create secret generic mongodb-credentials \
  --from-literal=username='genuser' \
  --from-literal=password='<strong-password>' \
  --dry-run=client -o yaml | kubectl apply -f -
```

### 4. Configure GitHub repository variables

In **GitHub → Settings → Secrets and variables → Actions**, configure:

```text
AWS_REGION=us-west-2
EKS_CLUSTER_NAME=<eks-cluster-name>
ECR_REPOSITORY=<ecr-repository-name>
AWS_ROLE_ARN=<github-actions-iam-role-arn>
```

### 5. Trigger the pipeline

```bash
git add .
git commit -m "Deploy Namagen to Amazon EKS"
git push origin main
```

GitHub Actions will scan, build, publish, and deploy the application.

---

## 🔎 Verify the Deployment

Check the Kubernetes resources:

```bash
kubectl -n namagen get deployments
kubectl -n namagen get statefulsets
kubectl -n namagen get pods -o wide
kubectl -n namagen get services
kubectl -n namagen get pvc
kubectl -n namagen get events --sort-by=.metadata.creationTimestamp
```

Check rollout status:

```bash
kubectl -n namagen rollout status deployment/namagen-app-deployment
kubectl -n namagen rollout status statefulset/mongodb-statefulset
```

Inspect application logs:

```bash
kubectl -n namagen logs deployment/namagen-app-deployment --tail=100
```

Inspect MongoDB logs:

```bash
kubectl -n namagen logs mongodb-statefulset-0 --tail=100
```

---

## 🌐 Access the Application

Retrieve the Network Load Balancer hostname:

```bash
kubectl -n namagen get service namagen-app-service
```

Or extract it directly:

```bash
export APP_URL="$(kubectl -n namagen get service namagen-app-service \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

echo "http://$APP_URL"
```

Open the returned URL in a browser.

---

## 💾 Persistence Validation

MongoDB stores its data on a PVC backed by Amazon EBS. To confirm that application data survives Pod replacement:

```bash
kubectl -n namagen get pvc
kubectl -n namagen delete pod mongodb-statefulset-0
kubectl -n namagen get pods -w
```

After the StatefulSet recreates the Pod, previously saved names should still exist because the EBS volume is reattached.

---

## 🧪 Useful Operational Commands

```bash
# Show the image currently deployed
kubectl -n namagen get deployment namagen-app-deployment \
  -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'

# Describe the application deployment
kubectl -n namagen describe deployment namagen-app-deployment

# Describe the MongoDB StatefulSet
kubectl -n namagen describe statefulset mongodb-statefulset

# Inspect the persistent volume claim
kubectl -n namagen describe pvc

# Follow application logs
kubectl -n namagen logs -f deployment/namagen-app-deployment
```

---

## 🛡️ Production Hardening Checklist

- [ ] Use GitHub OIDC instead of static AWS access keys
- [ ] Tag images with the Git commit SHA rather than only `latest`
- [ ] Block deployment when CodeQL or Trivy reports exceed the accepted severity threshold
- [ ] Store credentials in Kubernetes Secrets or AWS Secrets Manager
- [ ] Add application readiness and liveness probes
- [ ] Add MongoDB startup and readiness probes
- [ ] Define CPU and memory requests and limits
- [ ] Apply Kubernetes NetworkPolicies where supported
- [ ] Enable ECR image scanning and lifecycle policies
- [ ] Enable EKS control-plane and application logging
- [ ] Use encrypted EBS volumes
- [ ] Restrict the GitHub Actions IAM role to the minimum required permissions
- [ ] Add automated rollback or rollback documentation

---

## 🧭 Troubleshooting

### Application Pod cannot connect to MongoDB

```bash
kubectl -n namagen get service mongodb-service
kubectl -n namagen get endpoints mongodb-service
kubectl -n namagen get pods -l app=mongodb
```

Confirm that the service name in `MONGODB_URL` matches the MongoDB Service manifest and that the authentication database is correct.

### PVC remains in `Pending`

```bash
kubectl -n namagen describe pvc
kubectl get storageclass
kubectl -n kube-system get pods | grep ebs
```

Confirm that the Amazon EBS CSI Driver is installed and that the StorageClass is valid for the cluster and Availability Zone configuration.

### Load balancer hostname is not assigned

```bash
kubectl -n namagen describe service namagen-app-service
kubectl -n namagen get events --sort-by=.metadata.creationTimestamp
```

Confirm that the service type is `LoadBalancer` and that the cluster has the IAM and subnet configuration required to provision an AWS NLB.

### Deployment does not roll out

```bash
kubectl -n namegen rollout status deployment/namagen-app-deployment
kubectl -n namegen describe pod <pod-name>
kubectl -n namegen logs <pod-name> --previous
```

---

## 🎯 Engineering Outcomes

This project provides a portfolio-ready demonstration of:

- Building a secure software supply chain
- Operating containerized applications on Kubernetes
- Integrating GitHub Actions with AWS services
- Managing stateless and stateful workloads in the same EKS cluster
- Designing internal service discovery and public ingress
- Preserving database state across Pod recreation
- Applying practical DevSecOps controls before production deployment

---

## 📄 License

This project is available for educational and portfolio use. Add a `LICENSE` file to define the terms for redistribution and modification.

---

<div align="center">

Built with **GitHub Actions**, **Docker**, **Amazon ECR**, **Amazon EKS**, **Kubernetes**, **MongoDB**, and **Amazon EBS**.

</div>
