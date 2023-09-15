pipeline {
    // Agent needs to be a Linux environment with Docker CLI
    agent any

    options {
        timestamps()
        timeout(time: 10, unit: 'MINUTES')
    }

    environment {
        AWS_DEFAULT_REGION = 'il-central-1'
        ECR_REGISTRY = '644435390668.dkr.ecr.il-central-1.amazonaws.com/ec2-arthur'
        IMAGE_NAME = 'cowsay_multi'
        CONTAINER_PORT = '8080'
        DEPLOY_PORT = '80'
        STAGING_PORT = '3000'
        EC2_PUBLIC_DNS = 'ec2-51-16-152-185.il-central-1.compute.amazonaws.com'
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    def branch_name = env.BRANCH_NAME
                    sh "echo 'Pulling branch ${branch_name}'"
                    checkout scm
                }
            }
        }

        stage('Build') {
            steps {
                sh '''
                    docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Run') {
            when {
                expression {
                    return env.BRANCH_NAME =~ 'feature*'
                }
            }
            steps {
                sh '''
                    docker run -d --network=jenkins_default --name=${IMAGE_NAME} ${IMAGE_NAME}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Test') {
            when {
                expression {
                    return env.BRANCH_NAME =~ 'feature*'
                }
            }
            steps {
                retry(20) {
                    sleep(time: 3, unit: 'SECONDS')
                    sh '''
                        curl -f http://${IMAGE_NAME}:${CONTAINER_PORT}
                    '''
                }
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws_credentials',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        sh '''
                            /usr/bin/aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                            /usr/bin/aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                            /usr/bin/aws configure set region ${AWS_DEFAULT_REGION}
                            
                            docker login -u AWS -p $(aws ecr get-login-password --region ${AWS_DEFAULT_REGION}) ${ECR_REGISTRY}
                            docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${ECR_REGISTRY}:${BUILD_NUMBER}
                            docker push ${ECR_REGISTRY}:${BUILD_NUMBER}
                        '''
                    }
            }
        }

        stage('Deploy to EC2') {
            when {
                expression {
                    return env.BRANCH_NAME == 'master' || env.BRANCH_NAME == 'staging'
                }
            }
            steps {                    
                withCredentials([
                    sshUserPrivateKey(credentialsId: 'aws_key', keyFileVariable: 'SSH_KEY')
                ]) {      
                    sh """                                                
                        ssh -i \${SSH_KEY} admin@\${EC2_PUBLIC_DNS} "
                            echo 'Connected to EC2'
                            echo 'Logging into ECR through Docker'
                            docker login -u AWS -p \$(/usr/bin/aws ecr get-login-password --region \${AWS_DEFAULT_REGION}) \${ECR_REGISTRY}
                            echo 'Login successful'
                            docker pull \${ECR_REGISTRY}:\${BUILD_NUMBER}
                            
                            # Check and remove container
                            containerExists=\$(docker ps -a --format "{{.Names}}" | grep -w \${IMAGE_NAME})
                            if [ -n "\$containerExists" ]; then
                                docker stop \${IMAGE_NAME}
                                docker rm -f \${IMAGE_NAME}
                            fi
                            
                            if [[ "${env.BRANCH_NAME}" == 'master' ]]
                            then
                                echo " Master port: ${DEPLOY_PORT}"
                                docker run -d -p \${DEPLOY_PORT}:\${CONTAINER_PORT} --name \${IMAGE_NAME} \${ECR_REGISTRY}:\${BUILD_NUMBER}
                            else
                                echo " Staging port: ${STAGING_PORT}"
                                docker run -d -p \${STAGING_PORT}:\${CONTAINER_PORT} --name \${IMAGE_NAME} \${ECR_REGISTRY}:\${BUILD_NUMBER}
                            fi
                            
                            docker logout \${ECR_REGISTRY}
                        "
                    """      
                }
            }
        }

        stage('E2E Tests') {
            when {
                expression {
                    return env.BRANCH_NAME == 'master' || env.BRANCH_NAME == 'staging'
                }
            }
            steps {                                    
                withCredentials([
                    sshUserPrivateKey(credentialsId: 'aws_key', keyFileVariable: 'SSH_KEY')
                ]) {      
                    script {
                        if (env.BRANCH_NAME == 'master') {
                            echo "Branch ${env.BRANCH_NAME}"
                            echo "Master port: ${DEPLOY_PORT}"
                            sh "curl -f http://${EC2_PUBLIC_DNS}:${DEPLOY_PORT}"
                        } else {
                            sh """ ssh -i ${SSH_KEY} admin@\${EC2_PUBLIC_DNS} "
                                echo "Branch ${env.BRANCH_NAME}"                                
                                echo "Staging port: ${STAGING_PORT}"
                                curl -f http://${EC2_PUBLIC_DNS}:${STAGING_PORT}
                            "
                            """
                        }
                    }                                                     
                }
            }
        } 
    }

    post {
        always {
            sh '''
                docker stop ${IMAGE_NAME} || true
                docker rm --force ${IMAGE_NAME} || true
                docker image rm "${IMAGE_NAME}:${BUILD_NUMBER}" || true
                docker logout "${ECR_REGISTRY}"
            '''
            cleanWs()
        }

        failure {
            updateGitlabCommitStatus name: 'Jenkins', state: 'failed'
        }

        success {
            updateGitlabCommitStatus name: 'Jenkins', state: 'success'
        }
    }
}
