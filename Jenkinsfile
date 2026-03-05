pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        disableConcurrentBuilds()
    }

    triggers {
        githubPush()
    }

    environment {
        DOCKER_USER     = "vikasabhimanyu"
        DOCKER_CRED     = "dockerhub-creds"
        GIT_CRED        = "github-creds"
        KUBECONFIG_CRED = "kubeconfig-creds"
    }

    stages {

        stage('Skip CI Commit') {
            steps {
                script {
                    def msg = sh(
                        script: "git log -1 --pretty=%B",
                        returnStdout: true
                    ).trim()

                    if (msg.contains("[skip ci]")) {
                        echo "Skipping build triggered by Jenkins commit"
                        currentBuild.result = 'SUCCESS'
                        return
                    }
                }
            }
        }

        stage('Skip Jenkins Commit') {
            steps {
                script {
                    def author = sh(
                        script: "git log -1 --pretty=%an",
                        returnStdout: true
                    ).trim()

                    if (author == "jenkins") {
                        currentBuild.result = 'NOT_BUILT'
                        error("Commit created by Jenkins. Skipping pipeline.")
                    }
                }
            }
        }

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
                            error "Unsupported branch ${env.BRANCH_NAME}"
                    }

                    env.GIT_SHA = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()

                    env.BACKEND_IMAGE  = "${DOCKER_USER}/backend:${ENV}-${GIT_SHA}"
                    env.FRONTEND_IMAGE = "${DOCKER_USER}/frontend:${ENV}-${GIT_SHA}"
                }
            }
        }

        stage('Build & Push Docker Images') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CRED}",
                        usernameVariable: 'DOCKER_USER_VAR',
                        passwordVariable: 'DOCKER_PASS_VAR'
                    )
                ]) {
                    sh '''
                        echo $DOCKER_PASS_VAR | docker login -u $DOCKER_USER_VAR --password-stdin

                        docker build -t ${BACKEND_IMAGE} ./backend
                        docker push ${BACKEND_IMAGE}

                        docker build -t ${FRONTEND_IMAGE} ./frontend
                        docker push ${FRONTEND_IMAGE}
                    '''
                }
            }
        }

        stage('Update Image Tag in GitHub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${GIT_CRED}",
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_PASS'
                    )
                ]) {
                    sh '''
                        git config user.email "ci-bot@example.com"
                        git config user.name "jenkins"

                        git fetch origin
                        git checkout ${BRANCH_NAME}
                        git reset --hard origin/${BRANCH_NAME}

                        sed -i "s|image: vikasabhimanyu/backend:.*|image: ${BACKEND_IMAGE}|g" k8s/backend-deployment.yaml
                        sed -i "s|image: vikasabhimanyu/frontend:.*|image: ${FRONTEND_IMAGE}|g" k8s/frontend-deployment.yaml

                        git add k8s/*.yaml
                        git commit -m "Update image tags to ${GIT_SHA} [skip ci]" || echo "No changes"

                        git push https://${GIT_USER}:${GIT_PASS}@github.com/Vikas-Abhimanyu/k8s-assignment.git ${BRANCH_NAME}
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')
                ]) {
                    sh '''
                        export KUBECONFIG=$KUBECONFIG_FILE

                        kubectl apply -f k8s/backend-deployment.yaml -n ${NAMESPACE}
                        kubectl apply -f k8s/frontend-deployment.yaml -n ${NAMESPACE}

                        kubectl get pods -n ${NAMESPACE}

                        kubectl rollout status deployment/backend -n ${NAMESPACE}
                        kubectl rollout status deployment/frontend -n ${NAMESPACE}
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Deployment successful"
        }
        failure {
            echo "Deployment failed"
        }
    }
}
