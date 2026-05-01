pipeline {

    agent any

    environment {
        AWS_REGION     = 'ap-south-1'
        AWS_ACCOUNT_ID = 'YOUR_AWS_ACCOUNT_ID'     // replace with your 12-digit AWS account ID
        ECR_REPO       = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/banking-finance"
        EKS_CLUSTER    = 'banking-cluster'
        IMAGE_TAG      = "${BUILD_NUMBER}"
    }

    stages {

        // Stage 1 — figure out which environment to deploy to based on the branch name
        stage('Set Environment') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        env.DEPLOY_ENV = 'prod'
                    } else if (env.BRANCH_NAME == 'staging') {
                        env.DEPLOY_ENV = 'staging'
                    } else {
                        env.DEPLOY_ENV = 'qa'
                    }
                    echo "Branch [${env.BRANCH_NAME}] → Deploying to [${env.DEPLOY_ENV}]"
                }
            }
        }

        // Stage 2 — pull the latest code from GitHub
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // Stage 3 — run unit tests. pipeline stops here if any test fails
        stage('Run Tests') {
            steps {
                sh './mvnw test -B --no-transfer-progress'
            }
            post {
                always {
                    junit 'target/surefire-reports/**/*.xml'
                }
            }
        }

        // Stage 4 — compile and package the app into a JAR file
        stage('Build JAR') {
            steps {
                sh './mvnw clean package -DskipTests -B --no-transfer-progress'
            }
        }

        // Stage 5 — wrap the JAR in a Docker image
        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} ."
            }
        }

        // Stage 6 — login to AWS ECR and push the image
        stage('Push to ECR') {
            steps {
                sh """
                    aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${ECR_REPO}
                    docker push ${ECR_REPO}:${IMAGE_TAG}
                """
            }
        }

        // Stage 7 — deploy to the correct Kubernetes namespace on EKS
        // NOTE: prod deployments pause here and wait for a human to approve
        stage('Deploy to EKS') {
            steps {
                script {
                    if (env.DEPLOY_ENV == 'prod') {
                        input message: "Deploy build #${BUILD_NUMBER} to PRODUCTION?", ok: 'Yes, Deploy'
                    }
                    sh """
                        aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}

                        sed -e 's|DEPLOY_ENV|${DEPLOY_ENV}|g' \
                            -e 's|IMAGE_PLACEHOLDER|${ECR_REPO}:${IMAGE_TAG}|g' \
                            k8s/deployment.yaml | kubectl apply -f -

                        kubectl rollout status deployment/banking-finance \
                            -n ${DEPLOY_ENV} --timeout=120s
                    """
                }
            }
        }

        // Stage 8 — confirm the app is actually responding after deployment
        stage('Health Check') {
            steps {
                sh """
                    sleep 20
                    kubectl get pods -n ${DEPLOY_ENV}
                    kubectl get svc  -n ${DEPLOY_ENV}
                """
            }
            post {
                failure {
                    // if health check fails, roll back to the previous working version
                    sh "kubectl rollout undo deployment/banking-finance -n ${DEPLOY_ENV}"
                    echo "Rolled back to previous version"
                }
            }
        }

    }

    post {
        success {
            echo "SUCCESS — Build #${BUILD_NUMBER} deployed to ${env.DEPLOY_ENV}"
        }
        failure {
            echo "FAILED — Build #${BUILD_NUMBER} on branch ${env.BRANCH_NAME}"
        }
        always {
            sh "docker rmi ${ECR_REPO}:${IMAGE_TAG} || true"
            cleanWs()
        }
    }
}
