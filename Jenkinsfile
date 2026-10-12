pipeline {

    agent { label 'frontend' }   // pins the pipeline to the Jenkins worker node labeled "frontend"

    environment {
        AWS_REGION       = 'eu-north-1'
        AWS_ACCOUNT_ID   = credentials('aws-account-id')     // Jenkins "Secret text" credential

        ECR_REPOSITORY   = 'arms-frontend'
        EKS_CLUSTER      = 'arms-eks'
        NAMESPACE        = 'arms'

        ECR_REGISTRY     = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_IMAGE        = "${ECR_REGISTRY}/${ECR_REPOSITORY}"

        SONARQUBE_SERVER = 'SonarQube'
    }

    stages {

        // 1. Get application code from private GitHub repository
        stage('Checkout') {
            steps {
                git(
                    branch: 'main',
                    credentialsId: 'github-frontend',
                    url: 'git@github.com:YOUR_ORG/ARMS-Frontend.git'
                )
            }
        }

        // 2. Get Git commit ID and use it as image tag
        stage('Get Git Commit ID') {
            steps {
                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()

                    echo "Image Tag: ${env.IMAGE_TAG}"
                }
            }
        }

        // 3. Analyze source code
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh '''
                        sonar-scanner \
                        -Dsonar.projectKey=arms-frontend \
                        -Dsonar.projectName=ARMS-Frontend \
                        -Dsonar.sources=.
                    '''
                }
            }
        }

        // 4. Check SonarQube quality gate
        
     stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // 5. Build Docker image
        stage('Docker Build') {
            steps {
                sh """
                    docker build \
                    -t ${ECR_IMAGE}:${env.IMAGE_TAG} .
                """
            }
        }

        // 6. Scan Docker image
        stage('Trivy Image Scan') {
            steps {
                sh """
                    trivy image \
                    --severity HIGH,CRITICAL \
                    --exit-code 1 \
                    ${ECR_IMAGE}:${env.IMAGE_TAG}
                """
            }
        }

        // 7. Login to ECR — no withCredentials needed; the Jenkins EC2
        //    instance's IAM Role (via Instance Profile) supplies temporary
        //    AWS credentials automatically.
        stage('Login to ECR') {
            steps {
                sh """
                    aws ecr get-login-password \
                    --region ${AWS_REGION} | \
                    docker login \
                    --username AWS \
                    --password-stdin ${ECR_REGISTRY}
                """
            }
        }

        // 8. Push image to existing ECR repository
        stage('Push Image to ECR') {
            steps {
                sh """
                    docker push ${ECR_IMAGE}:${env.IMAGE_TAG}
                """
            }
        }

        // 9. Human approval before production deployment
        stage('Production Approval') {
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input(
                        message: "Deploy ${ECR_IMAGE}:${env.IMAGE_TAG} to production?",
                        ok: "Approve Deployment"
                    )
                }
            }
        }

        // 10. Configure kubectl access to EKS — same instance role, no
        //     withCredentials needed here either.
        stage('Configure EKS Access') {
            steps {
                sh """
                    aws eks update-kubeconfig \
                    --region ${AWS_REGION} \
                    --name ${EKS_CLUSTER}
                """
            }
        }

        // 11. Apply Kubernetes configuration — deployment.yaml is templated with
        //     envsubst so the image tag is injected in a single apply. This
        //     replaces the old "apply stale YAML, then kubectl set image" pattern:
        //     that caused two rollout events per run and left git's deployment.yaml
        //     permanently out of sync with what's actually running.
        //     deployment.yaml in git must reference the image as:
        //         image: ${ECR_IMAGE}:${IMAGE_TAG}
        stage('Apply Kubernetes Configuration') {
            steps {
                sh """
                    kubectl apply -f k8s/namespace.yaml

                    kubectl apply -f k8s/configmap.yaml -n ${NAMESPACE}

                    envsubst < k8s/deployment.yaml | kubectl apply -n ${NAMESPACE} -f -

                    kubectl apply -f k8s/service.yaml -n ${NAMESPACE}

                    kubectl apply -f k8s/ingress.yaml -n ${NAMESPACE}

                    kubectl apply -f k8s/hpa.yaml -n ${NAMESPACE}
                """
            }
        }

        // 12. Wait for rolling update to complete; roll back automatically on failure
        stage('Deployment Verification') {
            steps {
                script {
                    try {
                        sh """
                            kubectl -n ${NAMESPACE} \
                            rollout status deployment/arms-frontend \
                            --timeout=5m
                        """
                    } catch (err) {
                        echo "Rollout failed — rolling back to previous revision."
                        sh """
                            kubectl -n ${NAMESPACE} \
                            rollout undo deployment/arms-frontend
                        """
                        error("Deployment failed and was rolled back: ${err}")
                    }
                }
            }
        }

        // 13. Verify Kubernetes resources
        stage('Verify Deployment') {
            steps {
                sh """
                    kubectl -n ${NAMESPACE} get pods
                    kubectl -n ${NAMESPACE} get svc
                    kubectl -n ${NAMESPACE} get ingress
                """
            }
        }
    }

    post {

        success {
            echo "ARMS Frontend ${env.IMAGE_TAG} deployed successfully."
        }

        failure {
            echo "ARMS Frontend deployment failed."
        }

        always {
            echo "Jenkins pipeline completed."
        }
    }
}
