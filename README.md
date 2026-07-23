# Sample MERN with Microservices



For `helloService`, create `.env` file with the content:
```bash
PORT=3001
```

For `profileService`, create `.env` file with the content:
```bash
PORT=3002
MONGO_URL="specifyYourMongoURLHereWithDatabaseNameInTheEnd"
```

Finally install packages in both the services by running the command `npm install`.

<br/>
For frontend, you have to install and start the frontend server:

```bash
cd frontend
npm install
npm start
```

Note: This will run the frontend in the development server. To run in production, build the application by running the command `npm run build`

local test
```
docker build -t hello-service:v3.1 ./backend/helloService
docker run -p 3001:3001 -e PORT=3001 hello-service:v3.1

docker build -t profile-service:v3.1 ./backend/profileService
docker run -p 3002:3002 -e PORT=3002 -e MONGO_URL=mongodb+srv://tm_docker:Ri1OcidhdUoQdmTF@herovired-ppmcad.kncn7uu.mongodb.net/mernapp profile-service:v3.1

docker build -t frontend:v3.1 ./frontend
docker run -p 8080:80 frontend:v3.1


```

> ![alt text](image.png)

> ![alt text](image-1.png)


ECr docker images push
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

```