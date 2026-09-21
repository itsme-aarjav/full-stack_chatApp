pipeline {
    agent any

    tools {
        nodejs 'nodejs-18'
    }

    environment {
        DOCKERHUB_REPO_BACKEND  = "aarjavjainn/chatapp-backend"
        DOCKERHUB_REPO_FRONTEND = "aarjavjainn/chatapp-frontend"

        IMAGE_TAG = "build-${BUILD_NUMBER}"

        SONAR_SCANNER_HOME = tool "sonar-scanner"
    }

    stages {

        stage("Checkout") {
            steps {
                checkout scm
            }
        }

        stage("Frontend Lint") {
            steps {
                dir("frontend") {
                    sh '''
                        npm ci
                        npm run lint
                    '''
                }
            }
        }

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv("sonarqube") {
                    sh '''
                        "$SONAR_SCANNER_HOME/bin/sonar-scanner" \
                          -Dsonar.projectKey=chatapp \
                          -Dsonar.projectName=chatapp \
                          -Dsonar.sources=backend,frontend \
                          -Dsonar.exclusions="**/node_modules/**,**/dist/**,**/public/**"
                    '''
                }
            }
        }

        stage("Docker Build") {
            steps {
                sh '''
                    docker build \
                      -t "$DOCKERHUB_REPO_BACKEND:$IMAGE_TAG" \
                      ./backend

                    docker build \
                      -t "$DOCKERHUB_REPO_FRONTEND:$IMAGE_TAG" \
                      ./frontend
                '''
            }
        }

        stage("Trivy Scan") {
            steps {
                sh '''
                    trivy image \
                      --severity HIGH,CRITICAL \
                      --ignore-unfixed \
                      "$DOCKERHUB_REPO_BACKEND:$IMAGE_TAG"

                    trivy image \
                      --severity HIGH,CRITICAL \
                      --ignore-unfixed \
                      "$DOCKERHUB_REPO_FRONTEND:$IMAGE_TAG"
                '''
            }
        }

        stage("Docker Hub Push") {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "dockerhub-credentials",
                        usernameVariable: "DOCKERHUB_USERNAME",
                        passwordVariable: "DOCKERHUB_TOKEN"
                    )
                ]) {
                    sh '''
                        echo "$DOCKERHUB_TOKEN" | docker login \
                          --username "$DOCKERHUB_USERNAME" \
                          --password-stdin

                        docker push "$DOCKERHUB_REPO_BACKEND:$IMAGE_TAG"
                        docker push "$DOCKERHUB_REPO_FRONTEND:$IMAGE_TAG"

                        docker logout
                    '''
                }
            }
        }

        stage("Deploy with Helm") {
            steps {
                withCredentials([
                    string(
                        credentialsId: "chatapp-jwt-secret",
                        variable: "JWT_SECRET"
                    ),
                    string(
                        credentialsId: "chatapp-mongodb-username",
                        variable: "MONGO_USERNAME"
                    ),
                    string(
                        credentialsId: "chatapp-mongodb-password",
                        variable: "MONGO_PASSWORD"
                    )
                ]) {
                    sh '''
                        umask 077

                        cat > helm/chatapp/values-secret.yaml <<EOT
secrets:
  jwtSecret: "${JWT_SECRET}"
  mongodbUsername: "${MONGO_USERNAME}"
  mongodbPassword: "${MONGO_PASSWORD}"
  mongodbUri: "mongodb://${MONGO_USERNAME}:${MONGO_PASSWORD}@chatapp-mongodb:27017/chatApp?authSource=admin"
EOT

                        helm upgrade chatapp ./helm/chatapp \
                          --namespace chat-app \
                          --create-namespace \
                          --install \
                          --wait \
                          --timeout 5m \
                          -f helm/chatapp/values-secret.yaml \
                          --set backend.image.repository="$DOCKERHUB_REPO_BACKEND" \
                          --set backend.image.tag="$IMAGE_TAG" \
                          --set frontend.image.repository="$DOCKERHUB_REPO_FRONTEND" \
                          --set frontend.image.tag="$IMAGE_TAG"

                        rm -f helm/chatapp/values-secret.yaml
                    '''
                }
            }
        }

        stage("Verify Deployment") {
            steps {
                sh '''
                    kubectl rollout status deployment/chatapp-backend \
                      -n chat-app \
                      --timeout=120s

                    kubectl rollout status deployment/chatapp-frontend \
                      -n chat-app \
                      --timeout=120s

                    kubectl rollout status statefulset/chatapp-mongodb \
                      -n chat-app \
                      --timeout=120s

                    kubectl get pods -n chat-app
                    kubectl get hpa -n chat-app
                '''
            }
        }
    }

    post {
        always {
            sh '''
                rm -f helm/chatapp/values-secret.yaml
            '''
        }

        success {
            echo "ChatApp CI/CD pipeline completed successfully."
        }

        failure {
            echo "ChatApp CI/CD pipeline failed."
        }
    }
}
