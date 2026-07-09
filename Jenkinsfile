pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node23'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        ECR_REPO = '700918784883.dkr.ecr.ap-south-1.amazonaws.com/bookmyshow'
        AWS_REGION = 'ap-south-1'
        EKS_CLUSTER = 'bookmyshow-cluster'
        ANSIBLE_HOST = '172.31.14.157'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Akkhi-B/Book-My-Show.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=BookMyShow \
                    -Dsonar.projectKey=BookMyShow
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true,
                    credentialsId: 'sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                dir('bookmyshow-app') {
                    sh '''
                    npm install
                    '''
                }
            }
        }

        stage('OWASP Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./',
                odcInstallation: 'DP-Check'

                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy File Scan') {
            steps {
                sh '''
                trivy fs . > trivyfs.txt
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                docker build \
                -t bookmyshow:latest \
                -f bookmyshow-app/Dockerfile \
                bookmyshow-app
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin 700918784883.dkr.ecr.ap-south-1.amazonaws.com

                docker tag bookmyshow:latest $ECR_REPO:latest

                docker push $ECR_REPO:latest
                '''
            }
        }

        stage('Deploy using Ansible') {
            steps {
                sh '''
                ssh -o StrictHostKeyChecking=no ec2-user@$ANSIBLE_HOST << EOF

                cd ~/Book-My-Show

                aws eks update-kubeconfig \
                --region ap-south-1 \
                --name bookmyshow-cluster

                ansible-playbook deploy.yml

                EOF
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                ssh -o StrictHostKeyChecking=no ec2-user@$ANSIBLE_HOST << EOF

                kubectl get pods

                kubectl get svc

                EOF
                '''
            }
        }
    }

    post {

        always {

            archiveArtifacts artifacts: 'trivyfs.txt', fingerprint: true

        }

        success {

            echo 'Pipeline executed successfully.'

        }

        failure {

            echo 'Pipeline failed.'

        }
    }
}
