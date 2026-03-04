pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        DOCKER_USER     = "vikasabhimanyu"
        DOCKER_CRED     = "dockerhub-creds"
        KUBECONFIG_CRED = "kubeconfig-creds"
        GIT_CRED        = "github-creds"   // Jenkins credential for pushing back to GitHub
    }

    stages {

        stage('Determine Environment') {
            steps {
                script {
                    switch(env.BRANCH_NAME.toLowerCase()) {
                        case 'dev':
                            env.ENV = 'dev'
                            env.NAMESPACE = 'dev'
                            break
                        case 'testing':
                            env.ENV = 'testing'
                            env.NAMESPACE = 'testing'
                            break
                        case 'production':
                            env.ENV = 'production'
                            env.NAMESPACE = 'production'
                            break
                        default:
                            error "Unsupported branch: ${env.BRANCH_NAME}"
                    }

                    env.GIT_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()

                    env.BACKEND_IMAGE  = "${DOCKER_USER}/backend:${env.ENV}-${env.GIT_SHA}"
                    env.FRONTEND_IMAGE = "${DOCKER_USER}/frontend:${env.ENV}-${env.GIT_SHA}"

                    echo "Deploying to ${env.NAMESPACE} namespace with images:"
                    echo "Backend: ${env.BACKEND_IMAGE}"
                    echo "Frontend: ${env.FRONTEND_IMAGE}"
                }
            }
        }

        stage('Build & Push Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CRED}", 
                    usernameVariable: 'DOCKER_USER_VAR', passwordVariable: 'DOCKER_PASS_VAR')]) {
                    sh '''
                      echo $DOCKER_PASS_VAR | docker login -u $DOCKER_USER_VAR --password-stdin

                      echo "Building & pushing backend image..."
                      docker build -t ${BACKEND_IMAGE} ./backend
                      docker push ${BACKEND_IMAGE}

                      echo "Building & pushing frontend image..."
                      docker build -t ${FRONTEND_IMAGE} ./frontend
                      docker push ${FRONTEND_IMAGE}
                    '''
                }
            }
        }

        stage('Update Image Tag in GitHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${GIT_CRED}", 
                    usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                    sh '''
                      echo "Updating Deployment.yaml with new image tags..."
                      sed -i "s|image:.*|image: ${BACKEND_IMAGE}|g" k8s/backend-deployment.yaml
                      sed -i "s|image:.*|image: ${FRONTEND_IMAGE}|g" k8s/frontend-deployment.yaml

                      git config user.email "ci-bot@example.com"
                      git config user.name "CI Bot"
                      git add k8s/*.yaml
                      git commit -m "Update image tags to ${GIT_SHA}"
                      git push https://${GIT_USER}:${GIT_PASS}@github.com/Vikas-Abhimanyu/kubecoin-project.git HEAD:refs/heads/${BRANCH_NAME}

                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                      export KUBECONFIG=$KUBECONFIG_FILE
                      echo "Applying Kubernetes manifests..."
                      kubectl apply -f k8s/backend-deployment.yaml -n ${NAMESPACE}
                      kubectl apply -f k8s/frontend-deployment.yaml -n ${NAMESPACE}

                      echo "Waiting for rollout..."
                      if ! kubectl rollout status deployment/backend -n ${NAMESPACE}; then
                        echo "Backend rollout failed, rolling back..."
                        kubectl rollout undo deployment/backend -n ${NAMESPACE}
                        exit 1
                      fi

                      if ! kubectl rollout status deployment/frontend -n ${NAMESPACE}; then
                        echo "Frontend rollout failed, rolling back..."
                        kubectl rollout undo deployment/frontend -n ${NAMESPACE}
                        exit 1
                      fi
                    '''
                }
            }
        }

        stage('Post-Deploy Health Check') {
            steps {
                sh '''
                  echo "Running health checks..."
                  # Example: curl health endpoints
                  curl -f http://backend.${NAMESPACE}.svc.cluster.local:5000/health || exit 1
                  curl -f http://frontend.${NAMESPACE}.svc.cluster.local:8080/health || exit 1
                '''
            }
        }
    }

    post {
        failure {
            echo "Deployment failed. Check build logs and cluster status."
        }
    }
}

