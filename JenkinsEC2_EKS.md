# Jenkins EC2 Integration with EKS (mern-hello-profile)

This guide explains how to configure a Jenkins EC2 server to deploy applications to an Amazon EKS cluster using Helm.

---

## 1. Create IAM Role for Jenkins EC2

1. Go to **IAM → Roles → Create role**.
2. Choose **EC2** as the trusted entity.
3. Attach the following policies:
   - `AmazonEKSClusterPolicy`
   - `AmazonEKSWorkerNodePolicy`
   - `AmazonEC2ContainerRegistryFullAccess`
   - `CloudWatchFullAccess` (optional)
   - `AmazonSNSFullAccess` (or custom policy with `sns:Publish` to your topic ARN)
4. Name the role: **JenkinsEKSDeployRole**.

---

## 2. Attach IAM Role to Jenkins EC2

1. Go to **EC2 → Instances → Select Jenkins server → Actions → Security → Modify IAM role**.
2. Attach the role **JenkinsEKSDeployRole**.
3. Restart Jenkins EC2 if needed.

---

## 3. Map IAM Role into EKS RBAC

EKS requires mapping IAM identities to Kubernetes RBAC.

### Option A: Using eksctl
```bash
eksctl create iamidentitymapping \
  --cluster mern-hello-profile \
  --region ap-south-1 \
  --arn arn:aws:iam::129373676098:role/JenkinsEKSDeployRole \
  --username jenkins \
  --group system:masters
