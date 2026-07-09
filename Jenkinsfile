pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPO = '700918784883.dkr.ecr.ap-south-1.amazonaws.com/bookmyshow'
        ANSIBLE_HOST = '172.31.14.157'
        EKS_CLUSTER = 'bookmyshow-cluster'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Akkhi-B/Book-My-Show.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                docker build \
                  -t bookmyshow:latest \
                  -f bookmyshow-app/Dockerfile \
                  bookmyshow-app
                '''
            }
        }

        stage('Login to Amazon ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                    aws ecr get-login-password --region $AWS_REGION | \
                    docker login --username AWS --password-stdin 700918784883.dkr.ecr.ap-south-1.amazonaws.com
                    '''
                }
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                docker tag bookmyshow:latest $ECR_REPO:latest
                docker push $ECR_REPO:latest
                '''
            }
        }

        stage('Deploy using Ansible') {
            steps {
                sh '''
                ssh -o StrictHostKeyChecking=no ec2-user@$ANSIBLE_HOST <<EOF
                cd ~/Book-My-Show
                aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER
                ansible-playbook deploy.yml
                EOF
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                ssh -o StrictHostKeyChecking=no ec2-user@$ANSIBLE_HOST <<EOF
                kubectl get pods
                kubectl get svc
                EOF
                '''
            }
        }
    }

    post {
        success {
            echo 'Deployment Successful!'
        }

        failure {
            echo 'Deployment Failed!'
        }
    }
}
