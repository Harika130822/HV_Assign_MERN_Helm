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

Plugins (Docker pipeline, Amazon ECR, pipeline: AWS steps, Git, Github Integration / WebHook)

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


## secret from literal

```
kubectl create secret generic mongo-secret --from-literal=MONGO_URL='specifyYourMongoURLHereWithDatabaseNameInTheEnd'
kubectl get secret mongo-secret
kubectl describe secret mongo-secret
```

## Monitoring

```
ClusterName=streamingapp-cluster
RegionName=ap-south-1
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'
FluentBitReadFromTail='On'
FluentBitHttpServer='On'

curl -s https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml | \
sed 's/{{cluster_name}}/'${ClusterName}'/;s/{{region_name}}/'${RegionName}'/;s/{{http_server_toggle}}/"'${FluentBitHttpServer}'"/;s/{{http_server_port}}/"'${FluentBitHttpPort}'"/;s/{{read_from_head}}/"'${FluentBitReadFromHead}'"/;s/{{read_from_tail}}/"'${FluentBitReadFromTail}'"/' | \
kubectl apply -f -

kubectl get pods -n amazon-cloudwatch

kubectl delete pods -n amazon-cloudwatch --all

```
Verify in AWS Console: CloudWatch → Container Insights, and CloudWatch → Log groups → /aws/containerinsights/streamingapp-cluster/....

If pods are CrashLoopBackOff or no log groups appear, confirm the node role has CloudWatchAgentServerPolicy (Section 9), then:

## Helm

```
helm

helm lint .

helm template mernapp .

helm install mernapp . -n mern-helm --create-namespace

kubectl get all -n mern-helm

kubectl get pods -n mern-helm

```

> ![alt text](image-10.png)


```
PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm\mernapp> kubectl get nodes
NAME                                            STATUS   ROLES    AGE   VERSION
ip-192-168-40-78.ap-south-1.compute.internal    Ready    <none>   57m   v1.34.9-eks-8f14419
ip-192-168-76-193.ap-south-1.compute.internal   Ready    <none>   57m   v1.34.9-eks-8f14419
PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm\mernapp> kubectl get all -n mern-helm
NAME                                   READY   STATUS             RESTARTS   AGE
pod/frontend-6675d6475d-ckssp          0/1     ImagePullBackOff   0          9m11s
pod/hello-service-85f575fbbc-bjmf7     0/1     ImagePullBackOff   0          9m11s
pod/hello-service-85f575fbbc-ht4m9     0/1     ImagePullBackOff   0          9m11s
pod/profile-service-5466c6f744-7hps9   0/1     ImagePullBackOff   0          9m11s
pod/profile-service-5466c6f744-c8wjl   0/1     ImagePullBackOff   0          9m11s

NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/frontend          ClusterIP   10.100.120.37    <none>        80/TCP     9m11s
service/hello-service     ClusterIP   10.100.178.109   <none>        3001/TCP   9m11s
service/profile-service   ClusterIP   10.100.44.220    <none>        3002/TCP   9m11s

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/frontend          0/1     1            0           9m11s
deployment.apps/hello-service     0/2     2            0           9m11s
deployment.apps/profile-service   0/2     2            0           9m11s

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/frontend-6675d6475d          1         1         0       9m11s
replicaset.apps/hello-service-85f575fbbc     2         2         0       9m11s
replicaset.apps/profile-service-5466c6f744   2         2         0       9m11s

NAME                                                  REFERENCE                    TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/frontend          Deployment/frontend          cpu: <unknown>/80%   1         5         1          9m11s
horizontalpodautoscaler.autoscaling/hello-service     Deployment/hello-service     cpu: <unknown>/80%   1         5         2          9m11s
horizontalpodautoscaler.autoscaling/profile-service   Deployment/profile-service   cpu: <unknown>/80%   1         5         2          9m11s
PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm\mernapp> 


PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm\mernapp> kubectl get nodes
NAME                                            STATUS   ROLES    AGE   VERSION
ip-192-168-40-78.ap-south-1.compute.internal    Ready    <none>   57m   v1.34.9-eks-8f14419
ip-192-168-76-193.ap-south-1.compute.internal   Ready    <none>   57m   v1.34.9-eks-8f14419
PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm\mernapp> kubectl get all -n mern-helm
NAME                                   READY   STATUS             RESTARTS   AGE
pod/frontend-6675d6475d-ckssp          0/1     ImagePullBackOff   0          9m11s
pod/hello-service-85f575fbbc-bjmf7     0/1     ImagePullBackOff   0          9m11s
pod/hello-service-85f575fbbc-ht4m9     0/1     ImagePullBackOff   0          9m11s
pod/profile-service-5466c6f744-7hps9   0/1     ImagePullBackOff   0          9m11s
pod/profile-service-5466c6f744-c8wjl   0/1     ImagePullBackOff   0          9m11s

NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/frontend          ClusterIP   10.100.120.37    <none>        80/TCP     9m11s
service/hello-service     ClusterIP   10.100.178.109   <none>        3001/TCP   9m11s
service/profile-service   ClusterIP   10.100.44.220    <none>        3002/TCP   9m11s

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/frontend          0/1     1            0           9m11s
deployment.apps/hello-service     0/2     2            0           9m11s
deployment.apps/profile-service   0/2     2            0           9m11s

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/frontend-6675d6475d          1         1         0       9m11s
replicaset.apps/hello-service-85f575fbbc     2         2         0       9m11s
replicaset.apps/profile-service-5466c6f744   2         2         0       9m11s

NAME                                                  REFERENCE                    TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/frontend          Deployment/frontend          cpu: <unknown>/80%   1         5         1          9m11s
horizontalpodautoscaler.autoscaling/hello-service     Deployment/hello-service     cpu: <unknown>/80%   1         5         2          9m11s
horizontalpodautoscaler.autoscaling/profile-service   Deployment/profile-service   cpu: <unknown>/80%   1         5         2          9m11s
PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm\mernapp> 


PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm\mernapp> helm upgrade mernapp . -n mern-helm
Release "mernapp" has been upgraded. Happy Helming!
NAME: mernapp
LAST DEPLOYED: Thu Jul 23 20:33:52 2026
NAMESPACE: mern-helm
STATUS: deployed
REVISION: 2
NOTES:
1. Get the application URL by running these commands:
PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm\mernapp> 


PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm\mernapp> helm upgrade mernapp . -n mern-helm
Release "mernapp" has been upgraded. Happy Helming!
NAME: mernapp
LAST DEPLOYED: Thu Jul 23 20:33:52 2026
NAMESPACE: mern-helm
STATUS: deployed
REVISION: 2
NOTES:
1. Get the application URL by running these commands:
PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm\mernapp> 


PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm\mernapp> helm rollback mernapp 1 -n mern-helm
Rollback was a success! Happy Helming!
PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm\mernapp> kubectl get pods -n mern-helm       
NAME                               READY   STATUS             RESTARTS   AGE
frontend-6675d6475d-rkdlj          0/1     ImagePullBackOff   0          4m49s
hello-service-85f575fbbc-bjmf7     0/1     ImagePullBackOff   0          17m
hello-service-85f575fbbc-ht4m9     0/1     ImagePullBackOff   0          17m
profile-service-5466c6f744-7hps9   0/1     ImagePullBackOff   0          17m
profile-service-5466c6f744-c8wjl   0/1     ImagePullBackOff   0          17m
PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm\mernapp> 


PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm\mernapp> kubectl get all -n mern-helm
NAME                                   READY   STATUS    RESTARTS   AGE
pod/frontend-859c86c89-66z4w           1/1     Running   0          54s
pod/hello-service-f9ff4fb86-5kkvg      1/1     Running   0          54s
pod/profile-service-79cb5c46f7-9xs4d   1/1     Running   0          54s

NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/frontend          ClusterIP   10.100.120.37    <none>        80/TCP     19m
service/hello-service     ClusterIP   10.100.178.109   <none>        3001/TCP   19m
service/profile-service   ClusterIP   10.100.44.220    <none>        3002/TCP   19m

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/frontend          1/1     1            1           19m
deployment.apps/hello-service     1/1     1            1           19m
deployment.apps/profile-service   1/1     1            1           19m

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/frontend-6675d6475d          0         0         0       19m
replicaset.apps/frontend-859c86c89           1         1         1       54s
replicaset.apps/hello-service-85f575fbbc     0         0         0       19m
replicaset.apps/hello-service-f9ff4fb86      1         1         1       54s
replicaset.apps/profile-service-5466c6f744   0         0         0       19m
replicaset.apps/profile-service-79cb5c46f7   1         1         1       54s

NAME                                                  REFERENCE                    TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/frontend          Deployment/frontend          cpu: <unknown>/80%   1         5         1          19m
horizontalpodautoscaler.autoscaling/hello-service     Deployment/hello-service     cpu: <unknown>/80%   1         5         1          19m
horizontalpodautoscaler.autoscaling/profile-service   Deployment/profile-service   cpu: <unknown>/80%   1         5         1          19m
PS C:\Users\abhis\Harika\Assignments\HV_Assign_MERN_Helm\mernapp> 


```