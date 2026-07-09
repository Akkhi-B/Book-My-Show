pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node23'
    }

    environment {
        SCANNER_HOME = tool('sonar-scanner')
        AWS_REGION = 'ap-south-1'
        ECR_REPO = '700918784883.dkr.ecr.ap-south-1.amazonaws.com/bookmyshow'
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
                    sh """
                    ${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=BookMyShow \
                    -Dsonar.projectName=BookMyShow
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                dir('bookmyshow-app') {
                    sh 'npm install'
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck(
                    odcInstallation: 'DP-Check',
                    additionalArguments: '--scan ./'
                )

                dependencyCheckPublisher(
                    pattern: '**/dependency-check-report.xml'
                )
            }
        }

        stage('Trivy File Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
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
                aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 700918784883.dkr.ecr.ap-south-1.amazonaws.com

                docker tag bookmyshow:latest 700918784883.dkr.ecr.ap-south-1.amazonaws.com/bookmyshow:latest

                docker push 700918784883.dkr.ecr.ap-south-1.amazonaws.com/bookmyshow:latest
                '''
            }
        }

        stage('Deploy using Ansible') {
            steps {
                sh """
                ssh -o StrictHostKeyChecking=no ec2-user@${ANSIBLE_HOST} "cd ~/Book-My-Show && aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER} && ansible-playbook deploy.yml"
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                sh """
                ssh -o StrictHostKeyChecking=no ec2-user@${ANSIBLE_HOST} "kubectl get pods && kubectl get svc"
                """
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivyfs.txt', fingerprint: true
        }

        success {
            echo 'Pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed.'
        }
    }
}
