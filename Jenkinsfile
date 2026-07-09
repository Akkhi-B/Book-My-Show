pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node23'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        AWS_REGION = 'ap-south-1'
        ECR_REPO = '700918784883.dkr.ecr.ap-south-1.amazonaws.com/bookmyshow'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Source') {
            steps {
                git branch: 'main', url: 'https://github.com/staragile2016/Book-My-Show.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                    \$SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=BMS \
                    -Dsonar.projectKey=BMS
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
            }
        }

        stage('Install Dependencies') {
            steps {
                dir('bookmyshow-app') {
                    sh '''
                    rm -rf node_modules
                    npm install
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                docker build -t bookmyshow:latest \
                -f bookmyshow-app/Dockerfile \
                bookmyshow-app
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                    sh '''
                    aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 700918784883.dkr.ecr.ap-south-1.amazonaws.com

                    docker tag bookmyshow:latest \$ECR_REPO:latest

                    docker push \$ECR_REPO:latest
                    '''
                }
            }
        }

        stage('Deploy using Ansible') {
            steps {
                sh '''
                ssh -o StrictHostKeyChecking=no ec2-user@172.31.14.157 "cd ~/Book-My-Show && ansible-playbook deploy.yml"
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                ssh -o StrictHostKeyChecking=no ec2-user@172.31.14.157 "kubectl get pods && kubectl get svc"
                '''
            }
        }
    }

