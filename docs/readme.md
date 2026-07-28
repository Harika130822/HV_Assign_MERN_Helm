## local test
```
docker build -t hello-service:v3.1 ./backend/helloService
docker run -p 3001:3001 -e PORT=3001 hello-service:v3.1

docker build -t profile-service:v3.1 ./backend/profileService
docker run -p 3002:3002 -e PORT=3002 -e MONGO_URL=mongodb+srv://username:password@XYZ.mongodb.net/mernapp profile-service:v3.1

docker build -t frontend:v3.1 ./frontend
docker run -p 8080:80 frontend:v3.1


```

> ![alt text](image.png)

> ![alt text](image-1.png)


## ECR docker images push
```
aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws

docker tag profile-service:v3.1 public.ecr.aws/l5r9g0t4/mern/profile:v3.1
docker push public.ecr.aws/l5r9g0t4/mern/profile:v3.1

docker tag hello-service:v3.1 public.ecr.aws/l5r9g0t4/mern/hello:v3.1
docker push public.ecr.aws/l5r9g0t4/mern/hello:v3.1

docker tag frontend:v3.1 public.ecr.aws/l5r9g0t4/mern/frontend:v3.1
docker push public.ecr.aws/l5r9g0t4/mern/frontend:v3.1

```

> ![alt text](image-2.png)





## EC2 instance with Jenkins installed
```
sudo apt update -y

# Install Java (JDK 21)
sudo apt install -y openjdk-21-jdk

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

gpg --show-keys /etc/apt/keyrings/jenkins-keyring.asc

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update
sudo apt-get install fontconfig openjdk-21-jre
sudo apt-get install jenkins


sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

```

[Jenkins_Installation_Link(https://pkg.jenkins.io/debian-stable/?utm_source=copilot.com)]


> ![alt text](image-3.png)

> ![alt text](image-4.png)

```
# systemd service
sudo tee /etc/systemd/system/jenkins.service > /dev/null << 'EOF'

[Unit]
Description=Jenkins Continuous Integration Server
After=network.target

[Service]

User=jenkins
Group=jenkins


Environment="JENKINS_HOME=/var/lib/jenkins"

ExecStart=/path/to/java -jar /opt/jenkins/jenkins.war
#ExecStart=/usr/lib/jvm/java-21-openjdk-amd64/bin/java -jar /opt/jenkins/jenkins.war
Restart=on-failure
RestartSec=10


LimitNOFILE=8192

[Install]
WantedBy=multi-user.target
EOF

# systemctl
sudo systemctl daemon-reload
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins

```



## EC2 instance setup for docker, AWS CLI, Helm, kubectl

```
# Update packages
sudo apt-get update
# Install Docker
sudo apt-get install -y docker.io
# Enable and start Docker service
sudo systemctl enable docker
sudo systemctl start docker
# Add jenkins user to docker group (so Jenkins can run docker commands)
sudo usermod -aG docker jenkins
# make sure the jenkins user is in the docker group:

# Download AWS CLI v2 installer
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
# Unzip and install
unzip awscliv2.zip
sudo ./aws/install
# Verify


sudo apt install awscli -y
aws --version
#  Jenkins needs credentials. Either configure them under /var/lib/jenkins/.aws/credentials or inject via Jenkins credentials plugin.


# Download latest stable kubectl binary
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
# Make it executable and move to PATH
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
#For kubectl, ensure your kubeconfig (~/.kube/config) is readable by the jenkins user. - kubectl config: Place kubeconfig in /var/lib/jenkins/.kube/config and give ownership to jenkins.


# Download latest Helm script
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
# make sure the jenkins user has access to the kubeconfig and any required chart directories. - Helm: Needs access to the same kubeconfig as kubectl.



sudo -u jenkins docker --version
sudo -u jenkins aws --version
sudo -u jenkins kubectl version --client
sudo -u jenkins helm version

```

> ![alt text](image-9.png)


## Jenkins Plugin

Plugins (Docker pipeline, Amazon ECR, pipeline: AWS steps, SNS, AWS Credentials, Git, Github Integration / WebHook)

> ![alt text](image-8.png)

## Credentials

aws_credentials
MONGO_URL

> ![alt text](image-7.png)


## Pipeline to Build and Push Images

> ![alt text](image-5.png)

> ![alt text](image-6.png)


## Cluster EKS creation

```
eksctl create cluster --name mern-hello-profile --region ap-south-1 --nodegroup-name standard-workers --node-type t3.medium --nodes 2 --nodes-min 2 --nodes-max 3

aws eks --region ap-south-1 update-kubeconfig --name mern-hello-profile

kubectl config get-contexts

eksctl delete cluster --name mern-hello-profile --region ap-south-1



aws eks describe-nodegroup --cluster-name mern-hello-profile --nodegroup-name standard-workers --query "nodegroup.nodeRole" --region ap-south-1

aws iam attach-role-policy \
  --role-name <role-name-from-above> \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

```

> <img width="1572" height="913" alt="image" src="https://github.com/user-attachments/assets/d940b781-e74e-4adc-b5ab-19ff0a656696" />



## Monitoring

```
## Define variables
$ClusterName = "mern-hello-profile"
$RegionName = "ap-south-1"
$FluentBitHttpPort = "2020"
$FluentBitReadFromHead = "Off"
$FluentBitReadFromTail = "On"
$FluentBitHttpServer = "On"

# Download YAML
$yaml = Invoke-WebRequest -Uri "https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml"

# Perform replacements with single quotes around values
$content = $yaml.Content `
    -replace "{{cluster_name}}", "'$ClusterName'" `
    -replace "{{region_name}}", "'$RegionName'" `
    -replace "{{http_server_toggle}}", "'$FluentBitHttpServer'" `
    -replace "{{http_server_port}}", "'$FluentBitHttpPort'" `
    -replace "{{read_from_head}}", "'$FluentBitReadFromHead'" `
    -replace "{{read_from_tail}}", "'$FluentBitReadFromTail'"

# Save to file
$content | Out-File -FilePath "cwagent-fluent-bit-quickstart.yaml" -Encoding utf8

# Apply manifest
kubectl apply -f cwagent-fluent-bit-quickstart.yaml

kubectl delete -f cwagent-fluent-bit-quickstart.yaml


kubectl get pods -n amazon-cloudwatch

kubectl logs -n amazon-cloudwatch <cloudwatch-agent-pod>

kubectl delete pods -n amazon-cloudwatch --all


PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm> kubectl get ns
NAME                STATUS   AGE
amazon-cloudwatch   Active   14m
default             Active   35m
kube-node-lease     Active   35m
kube-public         Active   35m
kube-system         Active   35m
PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm> kubectl get all -n amazon-cloudwatch
NAME                         READY   STATUS    RESTARTS   AGE
pod/cloudwatch-agent-9s4hf   1/1     Running   0          8m38s
pod/cloudwatch-agent-r8dfs   1/1     Running   0          8m38s
pod/fluent-bit-bm9zh         1/1     Running   0          8m59s
pod/fluent-bit-jtdcz         1/1     Running   0          8m58s

NAME                              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/cloudwatch-agent   2         2         2       2            2           kubernetes.io/os=linux   14m
daemonset.apps/fluent-bit         2         2         2       2            2           kubernetes.io/os=linux   14m

```
Verify in AWS Console: CloudWatch → Container Insights, and CloudWatch → Log groups → /aws/containerinsights/streamingapp-cluster/....

If pods are CrashLoopBackOff or no log groups appear, confirm the node role has CloudWatchAgentServerPolicy (Section 9), then:

## Helm Initial Installation

```
helm

helm lint .

helm template mernapp .

helm install mernapp . -n mern-helm --create-namespace

kubectl get all -n mern-helm

kubectl get pods -n mern-helm

## secret from literal
kubectl create secret generic mongo-secret --from-literal=MONGO_URL='specifyYourMongoURLHereWithDatabaseNameInTheEnd' -n mern-helm
kubectl get secret mongo-secret  -n mern-helm
kubectl describe secret mongo-secret  -n mern-helm


PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm\mernapp> helm install mernapp . -n mern-helm --create-namespace
NAME: mernapp
LAST DEPLOYED: Sun Jul 26 18:39:02 2026
NAMESPACE: mern-helm
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm\mernapp> kubectl get all -n mern-helm
NAME                                  READY   STATUS    RESTARTS   AGE
pod/frontend-546d89f645-hdjxw         1/1     Running   0          5m53s
pod/hello-service-7cc896dcdb-d6lcv    1/1     Running   0          5m53s
pod/profile-service-65844654c-96jd6   1/1     Running   0          46s

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/frontend          ClusterIP   10.100.66.176   <none>        80/TCP     5m53s
service/hello-service     ClusterIP   10.100.250.33   <none>        3001/TCP   5m53s
service/profile-service   ClusterIP   10.100.36.171   <none>        3002/TCP   5m53s

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/frontend          1/1     1            1           5m53s
deployment.apps/hello-service     1/1     1            1           5m53s
deployment.apps/profile-service   1/1     1            1           5m53s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/frontend-546d89f645         1         1         1       5m53s
replicaset.apps/hello-service-7cc896dcdb    1         1         1       5m53s
replicaset.apps/profile-service-65844654c   1         1         1       5m53s

NAME                                                  REFERENCE                    TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/frontend          Deployment/frontend          cpu: 0%/80%   1         5         1          5m53s
horizontalpodautoscaler.autoscaling/hello-service     Deployment/hello-service     cpu: 1%/80%   1         5         1          5m53s
horizontalpodautoscaler.autoscaling/profile-service   Deployment/profile-service   cpu: 8%/80%   1         5         1          5m53s
PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm\mernapp> 
```
> <img width="1634" height="958" alt="image" src="https://github.com/user-attachments/assets/555093e5-dccb-4e34-9fc0-487e0820aba2" />

> <img width="1325" height="290" alt="image" src="https://github.com/user-attachments/assets/3ab25138-79c3-47a2-b78d-1637e7abf9f8" />


## Role Rolebinding

```
PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm> kubectl get serviceaccounts,roles,rolebindings -n mern-helm
NAME                        SECRETS   AGE
serviceaccount/default      0         35m
serviceaccount/jenkins-sa   0         7m9s

NAME                                           CREATED AT
role.rbac.authorization.k8s.io/helm-deployer   2026-07-28T12:32:48Z

NAME                                                          ROLE                 AGE
rolebinding.rbac.authorization.k8s.io/helm-deployer-binding   Role/helm-deployer   102s
```

## troubleshooting Tips
ImagePullBackOff,InvalidImageName,ErrorImagePull, CreateContainerConfigError
secrets are namespace scope 
