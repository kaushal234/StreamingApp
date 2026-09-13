# Deployment Runbook — StreamingApp

Command-by-command guide to deploy StreamingApp from scratch, both locally
(Minikube) and on AWS (ECR + EKS), plus CI, monitoring, and ChatOps. All AWS
commands use region `ap-south-1` and account `465708537536`; substitute your own
where noted.

---

## 0. Prerequisites

Install: Docker, kubectl, Helm 3+, Minikube (local), AWS CLI + eksctl (cloud).

```bash
docker --version
kubectl version --client
helm version
minikube version      # local path
aws --version         # cloud path
eksctl version        # cloud path
```

Fork the upstream repo and point your local clone at your fork:

```bash
# origin = your fork, upstream = original (to sync)
git remote add upstream https://github.com/UnpredictablePrashant/StreamingApp.git
```

---

## 1. Containerize & push images (Docker Hub)

Build all five images (note the differing build contexts) and push:

```bash
export DH=kaushalhub234

docker build -t $DH/streaming-auth:1.0.0 backend/authService
docker build -t $DH/streaming-stream:1.0.0 -f backend/streamingService/Dockerfile backend
docker build -t $DH/streaming-admin:1.0.0  -f backend/adminService/Dockerfile  backend
docker build -t $DH/streaming-chat:1.0.0   -f backend/chatService/Dockerfile   backend

# Frontend bakes API URLs in at build time — set them for the target Ingress host:
docker build \
  --build-arg REACT_APP_AUTH_API_URL=http://streamingapp.local/api \
  --build-arg REACT_APP_STREAMING_API_URL=http://streamingapp.local/api \
  --build-arg REACT_APP_STREAMING_PUBLIC_URL=http://streamingapp.local \
  --build-arg REACT_APP_ADMIN_API_URL=http://streamingapp.local/api/admin \
  --build-arg REACT_APP_CHAT_API_URL=http://streamingapp.local/api/chat \
  --build-arg REACT_APP_CHAT_SOCKET_URL=http://streamingapp.local \
  -t $DH/streaming-frontend:1.0.0 frontend

docker push $DH/streaming-auth:1.0.0
docker push $DH/streaming-stream:1.0.0
docker push $DH/streaming-admin:1.0.0
docker push $DH/streaming-chat:1.0.0
docker push $DH/streaming-frontend:1.0.0
```

Validate the frontend nginx config inside the image before pushing:
`docker run --rm $DH/streaming-frontend:1.0.0 nginx -t` → expect "test is successful".

---

## 2. Deploy locally (Minikube)

```bash
minikube start --driver=docker --cpus=4 --memory=3000
minikube addons enable ingress

# Install the whole stack with one command
helm install streamingapp ./streamingapp
kubectl get pods -w        # wait for all pods 1/1 Running (mongo first, then backends)

# Seed sample videos into the cluster's MongoDB
kubectl exec deploy/streamingapp-streaming -- npm run seed
```

Reach it in a browser:
1. Add to hosts file: `127.0.0.1 streamingapp.local`
2. In a dedicated terminal (leave running): `minikube tunnel`
3. Open **http://streamingapp.local** — register, login, browse, two-tab chat.

Verify:
```bash
kubectl get pods,svc,ingress
helm list          # streamingapp = deployed
```

---

## 3. AWS: push images to ECR

```bash
# Create one repository per component
for r in auth stream admin chat frontend; do
  aws ecr create-repository --repository-name streaming-$r --region ap-south-1
done

# Authenticate Docker to ECR
aws ecr get-login-password --region ap-south-1 \
  | docker login --username AWS --password-stdin 465708537536.dkr.ecr.ap-south-1.amazonaws.com

# Re-tag the local images to the ECR registry and push
export ECR=465708537536.dkr.ecr.ap-south-1.amazonaws.com
docker tag kaushalhub234/streaming-auth:1.0.0     $ECR/streaming-auth:1.0.0
docker tag kaushalhub234/streaming-stream:1.0.0   $ECR/streaming-stream:1.0.0
docker tag kaushalhub234/streaming-admin:1.0.0    $ECR/streaming-admin:1.0.0
docker tag kaushalhub234/streaming-chat:1.0.0     $ECR/streaming-chat:1.0.0
docker tag kaushalhub234/streaming-frontend:1.0.0 $ECR/streaming-frontend:1.0.0

for r in auth stream admin chat frontend; do docker push $ECR/streaming-$r:1.0.0; done
```

---

## 4. AWS: create the EKS cluster

```bash
eksctl create cluster --name streamingapp --region ap-south-1 \
  --nodes 2 --node-type t3.medium --managed
# ~15-20 min. Automatically updates kubeconfig.

kubectl get nodes                    # 2 nodes Ready
kubectl config current-context       # Administrator@streamingapp.ap-south-1.eksctl.io
```

### 4a. Storage (EBS CSI driver + default StorageClass)

EKS has no working default StorageClass out of the box, so the Mongo PVC stays
Pending until the EBS CSI driver + a CSI-backed StorageClass exist.

```bash
eksctl utils associate-iam-oidc-provider --region ap-south-1 --cluster streamingapp --approve

eksctl create iamserviceaccount --name ebs-csi-controller-sa --namespace kube-system \
  --cluster streamingapp --region ap-south-1 --role-name AmazonEKS_EBS_CSI_DriverRole \
  --role-only --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy --approve

eksctl create addon --name aws-ebs-csi-driver --cluster streamingapp --region ap-south-1 \
  --service-account-role-arn arn:aws:iam::465708537536:role/AmazonEKS_EBS_CSI_DriverRole --force

kubectl apply -f gp3-storageclass.yaml   # gp3 marked default, provisioner ebs.csi.aws.com
```

### 4b. Ingress controller (creates an AWS Load Balancer)

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace

kubectl get svc -n ingress-nginx     # note the EXTERNAL-IP (AWS ELB DNS name)
nslookup <elb-dns-name>              # resolve to an IP for the nip.io host
```

### 4c. Rebuild the frontend for the EKS host

The frontend must be rebuilt with the nip.io host baked in, then pushed to ECR:

```bash
docker build \
  --build-arg REACT_APP_AUTH_API_URL=http://streamingapp.<LB-IP>.nip.io/api \
  --build-arg REACT_APP_STREAMING_API_URL=http://streamingapp.<LB-IP>.nip.io/api \
  --build-arg REACT_APP_STREAMING_PUBLIC_URL=http://streamingapp.<LB-IP>.nip.io \
  --build-arg REACT_APP_ADMIN_API_URL=http://streamingapp.<LB-IP>.nip.io/api/admin \
  --build-arg REACT_APP_CHAT_API_URL=http://streamingapp.<LB-IP>.nip.io/api/chat \
  --build-arg REACT_APP_CHAT_SOCKET_URL=http://streamingapp.<LB-IP>.nip.io \
  -t $ECR/streaming-frontend:1.1.3 frontend
docker push $ECR/streaming-frontend:1.1.3
```

### 4d. Deploy the chart on EKS

```bash
helm install streamingapp ./streamingapp -f values-eks.yaml \
  --set ingress.host=streamingapp.<LB-IP>.nip.io \
  --set config.clientUrls="http://streamingapp.<LB-IP>.nip.io\,http://localhost:3000"

# point the frontend deployment at the rebuilt image (or set it in values-eks.yaml)
kubectl set image deploy/streamingapp-frontend frontend=$ECR/streaming-frontend:1.1.3

kubectl get pods                     # all 1/1 Running
kubectl exec deploy/streamingapp-streaming -- npm run seed
```

Open **http://streamingapp.\<LB-IP\>.nip.io** and run the full smoke test.

> **CORS note:** `config.clientUrls` must include the Ingress host, or auth
> rejects browser requests with a 500 (CORS). The comma is escaped as `\,` for
> Helm `--set`.

---

## 5. CI/CD (Jenkins)

- Create a **Pipeline** job → "Pipeline script from SCM" → Git →
  `https://github.com/kaushal234/StreamingApp.git`, branch `*/main`,
  Script Path `Jenkinsfile`.
- Add AWS credentials as Jenkins **Secret text** creds with IDs
  `jenkins-ecr-access-key-id` and `jenkins-ecr-secret-access-key`.
- Enable **Poll SCM** trigger: `H/5 * * * *`.
- **Build Now** → the pipeline builds all 5 images, pushes to ECR (tagged with
  the build number), and publishes a Slack notification via SNS.

---

## 6. Monitoring & Logging (CloudWatch)

```bash
# Give nodes CloudWatch permission, then install Container Insights
aws iam attach-role-policy --role-name <node-instance-role> \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
aws eks create-addon --cluster-name streamingapp \
  --addon-name amazon-cloudwatch-observability --region ap-south-1

# Example alarm
aws cloudwatch put-metric-alarm --alarm-name streamingapp-high-cpu \
  --namespace ContainerInsights --metric-name node_cpu_utilization \
  --dimensions Name=ClusterName,Value=streamingapp --statistic Average \
  --period 300 --threshold 80 --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 --region ap-south-1
```

Logs appear under `/aws/containerinsights/streamingapp/*`; metrics under
CloudWatch → Container Insights.

---

## 7. ChatOps (SNS → Chatbot → Slack)

```bash
aws sns create-topic --name streamingapp-deployments --region ap-south-1
```

Then in the **AWS Chatbot** console: authorize your Slack workspace, configure a
channel, and subscribe it to the `streamingapp-deployments` SNS topic. The
Jenkins pipeline publishes a Chatbot custom-notification-format message on each
build so it renders cleanly in Slack.

---

## 8. Teardown (stop billing)

```bash
eksctl delete cluster --name streamingapp --region ap-south-1     # cluster + nodes + LB

# clean up any orphaned EBS volume and the throwaway CI IAM user
aws ec2 describe-volumes --region ap-south-1 --filters Name=status,Values=available
aws ec2 delete-volume --volume-id <vol-id> --region ap-south-1
aws iam delete-user --user-name jenkins-ecr-push   # after detaching policies + deleting keys
```

Keep the ECR/Docker Hub images (deliverables, near-zero cost). Delete the
Jenkins credentials from the shared Jenkins instance.