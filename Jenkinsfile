pipeline {
    agent any

    environment {
        GIT_REPO         = "https://github.com/Harika130822/HV_Assign_MERN_Helm.git"
        AWS_REGION       = "ap-south-1"
        AWS_ACCOUNT_ID   = "129373676098"
        ECR_REGISTRY     = "public.ecr.aws/l5r9g0t4"
        IMAGE_TAG        = "${BUILD_NUMBER}"
        EKS_CLUSTER_NAME = "mern-hello-profile"
        MONGO_URL        = credentials('MONGO_URL')
        AWSSNS_TOPIC_ARN = "arn:aws:sns:ap-south-1:129373676098:jenkins-pipeline-alerts"
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main', url: "${GIT_REPO}"
            }
        }

        stage('Get Caller Identity') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws_credentials'
                ]]) {
                    sh 'aws sts get-caller-identity'
                }
            }
        }

        stage('Login to ECR Public') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws_credentials'
                ]]) {
                    sh '''
                        aws ecr-public get-login-password --region us-east-1 | \
                        docker login --username AWS --password-stdin public.ecr.aws
                    '''
                }
            }
        }

        stage('Build and Push Images') {
            steps {
                script {
                    def services = [
                        "backend/profileService": "mern/profile",
                        "backend/helloService"  : "mern/hello",
                        "frontend"              : "mern/frontend"
                    ]

                    services.each { folder, repoName ->

                        echo "Building image for ${folder}"

                        sh """
                            docker build -t ${repoName}:${IMAGE_TAG} ./${folder}
                            docker tag ${repoName}:${IMAGE_TAG} ${ECR_REGISTRY}/${repoName}:${IMAGE_TAG}
                            docker push ${ECR_REGISTRY}/${repoName}:${IMAGE_TAG}
                        """
                    }
                }
            }
        }

        stage('Update kubeconfig') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws_credentials'
                ]]) {
                    sh """
                        aws eks update-kubeconfig \
                        --region ${AWS_REGION} \
                        --name ${EKS_CLUSTER_NAME}
                    """
                }
            }
        }

        stage('Debug Helm Path') {
            steps {
                sh '''
                    pwd
                    ls -la
                    find . -name Chart.yaml
                '''
            }
        }

        stage('Deploy to EKS with Helm') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws_credentials'
                ]]) {
                    sh """
                        helm upgrade --install mernapp ./mernapp \
                        --set registry=${ECR_REGISTRY} \
                        --set tag=${IMAGE_TAG} \
                        -n mern-helm \
                        --create-namespace
                    """
                }
            }
        }
    }

    post {

        success {
            withCredentials([[
                $class: 'AmazonWebServicesCredentialsBinding',
                credentialsId: 'aws_credentials'
            ]]) {
                sh """
                    aws sns publish \
                    --topic-arn '${AWSSNS_TOPIC_ARN}' \
                    --subject 'Deployment Success' \
                    --message 'MERN application deployed successfully. Build Number: ${BUILD_NUMBER}'
                """
            }
        }

        failure {
            withCredentials([[
                $class: 'AmazonWebServicesCredentialsBinding',
                credentialsId: 'aws_credentials'
            ]]) {
                sh """
                    aws sns publish \
                    --topic-arn '${AWSSNS_TOPIC_ARN}' \
                    --subject 'Deployment Failed' \
                    --message 'MERN application deployment failed. Build Number: ${BUILD_NUMBER}'
                """
            }
        }

        always {
            cleanWs()
        }
    }
}
