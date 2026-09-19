pipeline {
    agent any

    options {
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_NAME  = "rahmanuddinmd17/multibranch-flask-app"
        GIT_USER    = "rahmanuddinmd"
        GIT_EMAIL   = "rahmanuddinmd@users.noreply.github.com"
        GITHUB_REPO = "rahmanuddinmd/Multi-Branch-Prod"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Check Commit') {
            when {
                branch 'main'
            }

            steps {
                script {
                    env.SKIP_PIPELINE = sh(
                        script: '''
                            if git log -1 --pretty=%B | grep -q "\\[skip ci\\]"; then
                                echo "true"
                            else
                                echo "false"
                            fi
                        ''',
                        returnStdout: true
                    ).trim()

                    echo "SKIP_PIPELINE=${env.SKIP_PIPELINE}"
                }
            }
        }

        stage('Build and Push Image') {
            when {
                allOf {
                    branch 'main'
                    expression {
                        env.SKIP_PIPELINE != 'true'
                    }
                }
            }

            steps {
                script {

                    env.IMAGE_TAG = "build-${BUILD_NUMBER}"

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'dockerhub-creds',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {

                        sh '''
                            set -e

                            echo "Building Docker image..."

                            docker build \
                                -t "${IMAGE_NAME}:${IMAGE_TAG}" .

                            echo "Logging into Docker Hub..."

                            echo "$DOCKER_PASS" | docker login \
                                -u "$DOCKER_USER" \
                                --password-stdin

                            echo "Pushing image..."

                            docker push "${IMAGE_NAME}:${IMAGE_TAG}"

                            echo "Image pushed successfully:"
                            echo "${IMAGE_NAME}:${IMAGE_TAG}"
                        '''
                    }
                }
            }
        }

        stage('Update K8s Manifest') {
            when {
                allOf {
                    branch 'main'
                    expression {
                        env.SKIP_PIPELINE != 'true'
                    }
                }
            }

            steps {
                script {

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'github-creds',
                            usernameVariable: 'GIT_USERNAME',
                            passwordVariable: 'GIT_TOKEN'
                        )
                    ]) {

                        sh '''
                            set -e

                            git config user.name "$GIT_USER"
                            git config user.email "$GIT_EMAIL"

                            echo "Updating local main branch..."

                            git fetch origin main
                            git checkout -B main origin/main

                            echo "Updating Kubernetes image..."

                            sed -i \
                            "s|image:.*|image: ${IMAGE_NAME}:${IMAGE_TAG}|" \
                            k8s/deployment.yaml

                            echo "Updated image:"
                            grep "image:" k8s/deployment.yaml

                            git add k8s/deployment.yaml

                            if git diff --cached --quiet; then

                                echo "No Kubernetes manifest changes."

                            else

                                git commit \
                                -m "Update image to ${IMAGE_TAG} [skip ci]"

                                echo "Pushing manifest update to GitHub..."

                                git push \
                                "https://${GIT_USERNAME}:${GIT_TOKEN}@github.com/${GITHUB_REPO}.git" \
                                main

                            fi
                        '''
                    }
                }
            }
        }
    }

    post {

        always {
            sh 'docker logout || true'
        }

        success {
            echo "Pipeline completed successfully."
        }

        failure {
            echo "Pipeline failed. Check Jenkins logs."
        }
    }
}
