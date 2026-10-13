pipeline {
    agent any

    environment {
        APP_NAME    = 'app-hello-nti'
        REPO_URL    = "https://github.com/Sara-Tarek1/Jenkins-repo.git"
        K8S_DIR     = 'K8s'
        NAMESPACE   = 'default'
    }

    stages {
        stage('Getting Repo files') {
            steps {
                git branch: "main", credentialsId: 'github', url: "${REPO_URL}"
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${APP_NAME}:${BUILD_NUMBER} ."
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD')]) {
                    sh """
                        echo "${DOCKER_PASSWORD}" | docker login -u "${DOCKER_USERNAME}" --password-stdin
                        docker tag ${APP_NAME}:${BUILD_NUMBER} ${DOCKER_USERNAME}/${APP_NAME}:${BUILD_NUMBER}
                        docker push ${DOCKER_USERNAME}/${APP_NAME}:${BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Setup Minikube Context') {
            steps {
                // kubeconfig is stored in Jenkins as a Secret File credential named
                // 'minikube-kubeconfig'. It is copied outside the workspace because
                // the workspace is wiped between builds.
                script {
                    withCredentials([file(credentialsId: 'minikube-kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                        sh """
                            mkdir -p ${env.TMPDIR ?: '/tmp'}/k8s
                            cp "\${KUBECONFIG_FILE}" ${env.TMPDIR ?: '/tmp'}/k8s/kubeconfig
                            chmod 600 ${env.TMPDIR ?: '/tmp'}/k8s/kubeconfig
                            export KUBECONFIG=${env.TMPDIR ?: '/tmp'}/k8s/kubeconfig
                            kubectl config current-context
                            kubectl get nodes
                        """
                    }
                }
            }
        }

        stage('Deploy to Minikube') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker',
                            usernameVariable: 'DOCKER_USERNAME',
                            passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh """
                            export KUBECONFIG=${env.TMPDIR ?: '/tmp'}/k8s/kubeconfig

                            kubectl apply -f ${K8S_DIR}/deployment.yaml
                            kubectl apply -f ${K8S_DIR}/service.yaml

                            kubectl set image deployment/${APP_NAME} ${APP_NAME}=${DOCKER_USERNAME}/${APP_NAME}:${BUILD_NUMBER}
                            kubectl rollout status deployment/${APP_NAME} --timeout=180s
                        """
                    }
                }
            }
        }

        stage('Verification') {
            steps {
                sh """
                    export KUBECONFIG=${env.TMPDIR ?: '/tmp'}/k8s/kubeconfig
                    echo "=== Pods ==="
                    kubectl get pods -o wide
                    echo "=== Services ==="
                    kubectl get services
                    echo "=== Deployment ==="
                    kubectl get deployment ${APP_NAME}
                """
            }
        }
    }

    post {
        always {
            echo "Build #${BUILD_NUMBER} of ${APP_NAME} finished with status: ${currentBuild.currentResult}"
        }
    }
}