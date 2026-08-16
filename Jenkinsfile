pipeline {
    agent any

    triggers {
        githubPush()
    }

    parameters {
        string(name: 'GIT_REPO_URL', defaultValue: 'https://github.com/Booket24/bookstore-api.git',
               description: 'bookstore-api source repository')
        string(name: 'GIT_BRANCH', defaultValue: 'master', description: 'Branch to build')
    }

    environment {
        AWS_REGION = 'eu-west-1'
        ECR_REPO_NAME  = 'bookstore-api'
        AWS_ACCOUNT_ID = credentials('aws-account-id')
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_TAG      = "${env.BUILD_NUMBER}"
        IMAGE_URI      = "${ECR_REGISTRY}/${ECR_REPO_NAME}:${IMAGE_TAG}"
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${params.GIT_BRANCH}",
                    url: "${params.GIT_REPO_URL}"
            }
        }

        stage('Build and Unit Test') {
            steps {
                sh 'mvn -B clean verify'
            }
            post {
                always {
                    junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${ECR_REPO_NAME}:${IMAGE_TAG} .
                    docker tag ${ECR_REPO_NAME}:${IMAGE_TAG} ${IMAGE_URI}
                    docker tag ${ECR_REPO_NAME}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO_NAME}:latest
                """
            }
        }

        stage('Scan Image') {
            steps {
                sh "trivy image --exit-code 1 --severity HIGH,CRITICAL ${ECR_REPO_NAME}:${IMAGE_TAG} || true"
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} \
                          | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        docker push ${IMAGE_URI}
                        docker push ${ECR_REGISTRY}/${ECR_REPO_NAME}:latest
                    """
                }
            }
        }

        stage('Deploy to Docker Hosts') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                    dir('ansible') {
                        sh """
                            ansible-playbook -i inventory.aws_ec2.yml deploy.yml \
                              -e image_uri=${IMAGE_URI} \
                              -e aws_region=${AWS_REGION}
                        """
                    }
                }
            }
        }

        stage('Smoke Test') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                    sh """
                        ALB_DNS=\$(aws elbv2 describe-load-balancers \
                          --names java-docker-app-alb \
                          --query 'LoadBalancers[0].DNSName' --output text --region ${AWS_REGION})
                        curl -fsS "http://\${ALB_DNS}/actuator/health"
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deployed ${IMAGE_URI} successfully."
        }
        failure {
            echo "Build or deploy failed. Check console output above."
        }
        always {
            sh 'docker image prune -f || true'
        }
    }
}
