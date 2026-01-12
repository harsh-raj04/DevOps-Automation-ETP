pipeline {
    agent any
    
    environment {
        AWS_REGION = 'us-east-1'
        TF_VAR_key_name = 'devops'
    }
    
    parameters {
        choice(name: 'ACTION', choices: ['apply', 'destroy'], description: 'Terraform action to perform')
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code...'
                checkout scm
            }
        }
        
        stage('Terraform Init') {
            steps {
                echo 'Initializing Terraform...'
                sh '''
                    terraform init
                '''
            }
        }
        
        stage('Terraform Plan') {
            steps {
                echo 'Planning Terraform changes...'
                sh '''
                    terraform plan -out=tfplan
                '''
            }
        }
        
        stage('Terraform Apply') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                echo 'Applying Terraform configuration...'
                sh '''
                    terraform apply -auto-approve tfplan
                '''
            }
        }
        
        stage('Wait for Instance Ready') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                script {
                    echo 'Retrieving instance ID...'
                    def instanceId = sh(
                        script: 'terraform output -raw instance_id',
                        returnStdout: true
                    ).trim()
                    
                    echo "Instance ID: ${instanceId}"
                    echo 'Waiting for instance to be ready...'
                    
                    // Wait for instance status checks to pass
                    sh """
                        aws ec2 wait instance-status-ok \\
                            --instance-ids ${instanceId} \\
                            --region ${AWS_REGION}
                    """
                    
                    echo 'Instance is ready!'
                }
            }
        }
        
        stage('Verify Web Service') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                script {
                    echo 'Retrieving instance public IP...'
                    def publicIp = sh(
                        script: 'terraform output -raw instance_public_ip',
                        returnStdout: true
                    ).trim()
                    
                    echo "Public IP: ${publicIp}"
                    
                    // Wait a bit for SSH to be fully ready and web service to start
                    echo 'Waiting for SSH and web service to be ready...'
                    sleep(time: 30, unit: 'SECONDS')
                    
                    // SSH into instance and verify web service
                    echo 'Verifying web service via SSH...'
                    sh """
                        # Set proper permissions for SSH key
                        chmod 600 ~/.ssh/devops.pem
                        
                        # SSH and run curl to check web service
                        ssh -i ~/.ssh/devops.pem \\
                            -o StrictHostKeyChecking=no \\
                            -o UserKnownHostsFile=/dev/null \\
                            ec2-user@${publicIp} \\
                            'curl -s -o /dev/null -w "%{http_code}" http://localhost'
                    """
                    
                    def httpStatus = sh(
                        script: """
                            ssh -i ~/.ssh/devops.pem \\
                                -o StrictHostKeyChecking=no \\
                                -o UserKnownHostsFile=/dev/null \\
                                ec2-user@${publicIp} \\
                                'curl -s -o /dev/null -w "%{http_code}" http://localhost'
                        """,
                        returnStdout: true
                    ).trim()
                    
                    echo "HTTP Status Code: ${httpStatus}"
                    
                    if (httpStatus == '200') {
                        echo '✓ Web service is returning 200 OK - SUCCESS!'
                    } else {
                        error("Web service returned ${httpStatus} instead of 200")
                    }
                    
                    // Also test from outside
                    echo 'Testing web service from external access...'
                    sh "curl -I http://${publicIp}"
                }
            }
        }
        
        stage('Terraform Destroy') {
            when {
                expression { params.ACTION == 'destroy' }
            }
            steps {
                echo 'Destroying Terraform resources...'
                sh '''
                    terraform destroy -auto-approve
                '''
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completed successfully!'
            script {
                if (params.ACTION == 'apply') {
                    def publicIp = sh(
                        script: 'terraform output -raw instance_public_ip 2>/dev/null || echo "N/A"',
                        returnStdout: true
                    ).trim()
                    if (publicIp != 'N/A') {
                        echo "Web server is accessible at: http://${publicIp}"
                    }
                }
            }
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
