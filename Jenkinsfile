pipeline {
    agent {
        label 'TRAVEL'
    }

    environment {
        GIT_REPO = 'https://github.com/ChandulaJ/TravelTrails-MERN-stack-app.git' 
        DOCKER_COMPOSE_REPO = 'https://github.com/ChandulaJ/TravelTrails-manifest.git'
        INSTANCE_IP = '100.29.182.72'
        SCANNER_HOME = tool 'sonar-tool'
    }

    stages {
        stage('Clean workspace') {
            steps {
                cleanWs()
            }
        }
        
        
        stage('Clone Repository with poll enabled') {
            steps {
                git branch: 'main', url: "${GIT_REPO}"
            }
        }
        
        // stage('Sonarqube Scan') {
        //     steps {
        //         withSonarQubeEnv('sonar-scanner') {
        //             sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=test2 \
        //             -Dsonar.java.binaries=. \
        //             -Dsonar.projectKey=test2 '''
        //         }
        //     }
        // }


        stage('Navigate to location of docker compose file') {
            steps { 
                sh 'cd /opt/jenkins-slave/workspace/TravelTrails/TravelTrails && ls -la'
            }
        }

        stage('Remove existing Docker Images') {
            steps {
                sh 'cd /opt/jenkins-slave/workspace/TravelTrails/TravelTrails && docker-compose down'
                sh 'sudo docker rmi mongo:4.4.25 traveltrails_frontend traveltrails_backend --force'
                sh 'sudo docker system prune -f'
            }
        }

        stage('Build Docker Images') {
            steps {
                sh 'cd /opt/jenkins-slave/workspace/TravelTrails/TravelTrails && docker-compose up --build -d'
            }
        }

        stage('Upload Container Image to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    sh "docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}"
                    sh "docker tag traveltrails_frontend ${DOCKER_USERNAME}/traveltrails_frontend:latest"
                    sh "docker tag traveltrails_backend ${DOCKER_USERNAME}/traveltrails_backend:latest"
                    sh "docker push ${DOCKER_USERNAME}/traveltrails_frontend:latest"
                    sh "docker push ${DOCKER_USERNAME}/traveltrails_backend:latest"
                }
            }
        }


   
        // stage('Check EC2 Instance with Tag') {
        //     steps {
        //         script {
        //             try {
        //                 def awsCliOutput = sh(script: 'aws ec2 describe-instances --filters "Name=tag:Name,Values=DeployServer" --query "Reservations[*].Instances[*].InstanceId" --output text', returnStdout: true).trim()
                        
        //                 if (awsCliOutput) {
        //                     echo "EC2 instance(s) with tag 'Name=DeployServer' found. Skipping Terraform stages."
        //                     env.INSTANCE_EXISTS = 'true'
        //                 } else {
        //                     echo "No EC2 instance with tag 'Name=DeployServer' found. Proceeding with Terraform stages."
        //                     env.INSTANCE_EXISTS = 'false'
        //                 }
        //             } catch (Exception e) {
        //                 env.INSTANCE_EXISTS = 'false'
        //                 error("Error while running AWS CLI command: ${e.message}")
        //             }
        //         }
        //     }
        // }


        stage('Terraform Init') {
            //   when {
            //     expression { env.INSTANCE_EXISTS == 'false' }
            // }
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'AWS-AMI']]) {
                    // Initialize Terraform
                    sh 'terraform init'
                }
            }
        }

        stage('Terraform Plan') {
            //   when {
            //     expression { env.INSTANCE_EXISTS == 'false' }
            // }
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'AWS-AMI']]) {
                    // Plan the Terraform deployment
                    sh 'terraform plan -out=tfplan'
                }
            }
        }

        stage('Terraform Apply') {
            //   when {
            //     expression { env.INSTANCE_EXISTS == 'false' }
            // }
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'AWS-AMI']]) {
                    // Apply the Terraform plan to create the EC2 instance
                    sh 'terraform apply -input=false tfplan'
                }
            }
        }
        
        stage('Terraform Refresh') {
            //   when {
            //     expression { env.INSTANCE_EXISTS == 'false' }
            // }
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'AWS-AMI']]) {
                    sh 'terraform refresh'
                }
            }
        }
        
        
        stage('Get EC2 Instance IP') {
            steps {
                script {
                    try {
                        env.INSTANCE_IP = sh(script: 'terraform output -raw instance_ip', returnStdout: true).trim()
                        echo "EC2 Instance IP: ${env.INSTANCE_IP}"
                    } catch (Exception e) {
                        error("Failed to get EC2 instance IP. Please check the Terraform output. Error: ${e.message}")
                    }
                }
            }
        }

    // stage('Wait for EC2 Instance to be Ready') {
    //         steps {
    //             script {
    //                 def instanceIp = "34.204.105.197"
    //                 def maxRetries = 10
    //                 def retryCount = 0
    //                 def isInstanceUp = false

    //                 while (!isInstanceUp && retryCount < maxRetries) {
    //                     try {
    //                         echo "Checking if instance is up (Attempt ${retryCount + 1})..."
    //                         sh "nc -zv ${instanceIp} 22"
    //                         isInstanceUp = true
    //                         echo "Instance is up!"
    //                     } catch (Exception e) {
    //                         echo "Instance is not up yet. Retrying in 30 seconds..."
    //                         sleep(30)
    //                         retryCount++
    //                     }
    //                 }

    //                 if (!isInstanceUp) {
    //                     error "Instance did not become ready in time."
    //                 }
    //             }
    //         }
    //     }


        // stage('Install Docker using Ansible') {
        //     steps {
        //         script {
        //             sleep 20
        //             def instanceIp = "34.204.105.197"
        //             if (instanceIp) {
        //                 writeFile file: 'inventory', text: "[deploy_server]\n${instanceIp} ansible_ssh_private_key_file=~/.ssh/jenkins-sanjula.pem ansible_user=ubuntu"
        //                 sh 'ansible-playbook -i inventory ansible-playbook.yml'
        //             } else {
        //                 echo "No instance created, skipping Ansible playbook execution."
        //             }
        //         }
        //     }
        // }

          stage('Wait for EC2 Instance to be Ready') {
            // when {
            //     expression { env.INSTANCE_EXISTS == 'false' }
            // }
            steps {
                script {
                    echo "Waiting for EC2 instance to be ready..."
                    sleep(time: 10, unit: 'SECONDS')
                }
            }
        }
        

 stage('Install Docker using Ansible') {
            // when {
            //     expression { env.INSTANCE_EXISTS == 'false' }
            // }
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'traveltrails-deploy-server', keyFileVariable: 'SSH_KEY_PATH')]) {
                    ansiblePlaybook(
                        playbook: 'ansible-playbook.yml',
                        inventory: "${env.INSTANCE_IP},",
                        credentialsId: 'traveltrails-deploy-server',
                        extras: "-u ubuntu --private-key \${SSH_KEY_PATH}"
                    )
                }
            }
        }


// stage('Pull Docker Image from Docker Hub and Run') {
//     steps {
//             script {
//                 def DEPLOYMENT_SERVER = "${env.INSTANCE_IP}"
//                 if (DEPLOYMENT_SERVER) {
//                     // Clone Docker Compose repository
//                     sh """
//                         ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ${SSH_USER}@${DEPLOYMENT_SERVER} git clone https://github.com/ChandulaJ/TravelTrails-manifest.git /tmp/docker-compose
//                     """
                    
//                     // Navigate to directory and run Docker Compose
//                     sh """
//                         ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ${SSH_USER}@${DEPLOYMENT_SERVER} cd /tmp/docker-compose && docker-compose up -d
//                     """
//                 } else {
//                     error "Deployment server IP not found."
//                 }
//             }
//     }
// }

      
   }
    
    
    
    
    post {
        always {
            script {

                def userInput = input(
                    id: 'ProceedDestroy', message: 'Do you want to destroy the deployment server?', parameters: [
                    [$class: 'BooleanParameterDefinition', defaultValue: true, description: '', name: 'Proceed']
                ])
                if (userInput) {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'AWS-AMI']]) {
                        sh 'terraform destroy -auto-approve'
                    }
                } else {
                    echo "Destruction aborted by user"
                }
            }
        }
    }
    


}
