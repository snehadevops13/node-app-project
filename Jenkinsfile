pipeline {
    
    agent any

    parameters {
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Docker image tag')
    }
    
    environment {
        ECR_VOTE_REPO = '064057453467.dkr.ecr.us-east-1.amazonaws.com/node-app-vote-repo'
        ECR_RESULT_REPO = '064057453467.dkr.ecr.us-east-1.amazonaws.com/node-app-result-repo'
        ECR_WORKER_REPO = '064057453467.dkr.ecr.us-east-1.amazonaws.com/node-app-worker-repo'
        AWS_REGION = 'us-east-1'
    }
    
    stages {
        stage('Checkout') {
            steps {
               git branch: 'main', credentialsId: 'JenkinsToGitHub', url: 'https://github.com/snehadevops13/node-app-project.git'
            }
        } 
        
        stage('Build and Push Result Docker Image') {
           
            steps {
                script {
                    dir('result') {  // Navigate to the Node.js app folder
                        def image = docker.build("${ECR_RESULT_REPO}:${params.IMAGE_TAG}")
                        
                        // Login to ECR (assumes AWS credentials are configured in Jenkins)
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_RESULT_REPO}"
                        
                        // Push to ECR
                        image.push()
                    }
                }
            }
        }

        stage('Build and Push Vote Docker Image') {
            steps {
                script {
                    dir('vote') {  // Navigate to the Node.js app folder
                        def image = docker.build("${ECR_VOTE_REPO}:${params.IMAGE_TAG}")
                        
                        // Login to ECR (assumes AWS credentials are configured in Jenkins)
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_VOTE_REPO}"
                        
                        // Push to ECR
                        image.push()
                    }
                }
            }
        }

        stage('Build and Push Worker Docker Image') {
            steps {
                script {
                    dir('worker') {  // Navigate to the Node.js app folder
                        def image = docker.build("${ECR_WORKER_REPO}:${params.IMAGE_TAG}")
                        
                        // Login to ECR (assumes AWS credentials are configured in Jenkins)
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_WORKER_REPO}"
                        
                        // Push to ECR
                        image.push()
                    }
                }
            }
                     
        }
        stage('Deploy to Remote Docker Host') {
    steps {
        script {
            sshagent(['JenkinsToApp']) {
                sh """
                ssh -o StrictHostKeyChecking=no ubuntu@10.0.3.46 bash -s << 'ENDSSH'

                set -e

                export IMAGE_TAG="${IMAGE_TAG}"
                export ECR_VOTE_REPO="${ECR_VOTE_REPO}"
                export ECR_RESULT_REPO="${ECR_RESULT_REPO}"
                export ECR_WORKER_REPO="${ECR_WORKER_REPO}"

                echo "Deploying with tag: \$IMAGE_TAG"

                docker network create front-tier 2>/dev/null || true 
                docker network create back-tier 2>/dev/null || true 
                docker volume create db-data 2>/dev/null || true 

                # -------------------------------
                # FUNCTION: stop container safely
                # -------------------------------
                stop_container () {
                  if [ \$(docker ps -a --format '{{.Names}}' | grep -w \$1 | wc -l) -gt 0 ]; then
                      echo "Stopping old container: \$1"
                      docker stop \$1
                      docker rm \$1
                  else
                      echo "Container \$1 not found, skipping..."
                  fi
                }

                # Stop only project containers
                stop_container redis
                stop_container db
                stop_container vote-container
                stop_container result-container
                stop_container worker-container

                # -------------------------------
                # Pull latest images
                # -------------------------------
                docker pull \${ECR_VOTE_REPO}:\${IMAGE_TAG}
                docker pull \${ECR_RESULT_REPO}:\${IMAGE_TAG}
                docker pull \${ECR_WORKER_REPO}:\${IMAGE_TAG}

                # -------------------------------
                # Run infrastructure containers
                # -------------------------------
                docker run -d --name redis --network back-tier \
                  -v "\$HOME/healthchecks:/healthchecks" \
                  --health-cmd "/healthchecks/redis.sh" \
                  --health-interval 5s redis:5.0-alpine3.10

                docker run -d --name db --network back-tier \
                  -e POSTGRES_USER=postgres \
                  -e POSTGRES_PASSWORD=postgres \
                  -e POSTGRES_HOST_AUTH_METHOD=trust \
                  -v db-data:/var/lib/postgresql/data \
                  -v "\$HOME/healthchecks:/healthchecks" \
                  --health-cmd "/healthchecks/postgres.sh" \
                  --health-interval 5s postgres:9.4

                # -------------------------------
                # Run application containers
                # -------------------------------
                docker run -d --name vote-container -p 8081:80 \
                  --network front-tier --network back-tier \
                  \${ECR_VOTE_REPO}:\${IMAGE_TAG}

                docker run -d --name result-container -p 8082:80 \
                  --network front-tier --network back-tier \
                  \${ECR_RESULT_REPO}:\${IMAGE_TAG}

                docker run -d --name worker-container \
                  --network back-tier \
                  \${ECR_WORKER_REPO}:\${IMAGE_TAG}

                echo "Deployment completed successfully "

ENDSSH
                """
                    }
                }
            }
        }

        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }
          
    }

}
