pipeline {
    agent any
    
    environment {
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        TF_IN_AUTOMATION      = '1'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Terraform Init') {
            steps {
                sh 'terraform init'
                script {
                    def initStatus = sh(script: 'terraform init', returnStatus: true)
                    if (initStatus != 0) {
                        error "Terraform initialization failed"
                    }
                }
            }
        }
        
        stage('Terraform Plan') {
            steps {
                sh 'terraform plan -out=tfplan'
                script {
                    def planStatus = sh(script: 'terraform plan -out=tfplan', returnStatus: true)
                    if (planStatus != 0) {
                        error "Terraform plan failed"
                    }
                }
            }
        }
        
        stage('Approval') {
            steps {
                input message: 'Do you want to apply the terraform plan?'
            }
        }
        
        stage('Terraform Apply') {
            steps {
                sh 'terraform apply -auto-approve tfplan'
                script {
                    def applyStatus = sh(script: 'terraform apply -auto-approve tfplan', returnStatus: true)
                    if (applyStatus != 0) {
                        error "Terraform apply failed"
                    }
                }
            }
        }
        
        stage('Verify Resources') {
            steps {
                script {
                    // Verify VPC
                    def vpcExists = sh(script: '''
                        aws ec2 describe-vpcs --filters "Name=tag:Name,Values=main-vpc" --query "Vpcs[*].VpcId" --output text
                    ''', returnStatus: true)
                    if (vpcExists != 0) {
                        error "VPC verification failed"
                    }
                    
                    // Verify Public Instance
                    def publicInstanceExists = sh(script: '''
                        aws ec2 describe-instances --filters "Name=tag:Name,Values=public-instance" --query "Reservations[*].Instances[*].InstanceId" --output text
                    ''', returnStatus: true)
                    if (publicInstanceExists != 0) {
                        error "Public instance verification failed"
                    }
                    
                    // Verify NGINX
                    def publicIP = sh(script: '''
                        aws ec2 describe-instances --filters "Name=tag:Name,Values=public-instance" --query "Reservations[*].Instances[*].PublicIpAddress" --output text
                    ''', returnStatus: true)
                    def nginxStatus = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://${publicIP}", returnStatus: true)
                    if (nginxStatus != 200) {
                        error "NGINX verification failed"
                    }
                }
            }
        }
    }
    
    post {
        failure {
            sh 'terraform destroy -auto-approve'
        }
    }
}