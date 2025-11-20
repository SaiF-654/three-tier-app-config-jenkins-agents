pipeline {
    agent any  // agent-dev | agent-stg | agent-prod

    environment {
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-credentials'
        BRANCH_NAME = "${env.BRANCH_NAME}"
        IMAGE_TAG   = "${BRANCH_NAME}-${BUILD_NUMBER}"
    }

    options {
        skipStagesAfterUnstable()
        ansiColor('xterm')
    }

    stages {
        stage('Checkout Source') {
            steps { checkout scm }
        }

        stage('Docker Hub Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKERHUB_CREDENTIALS_ID}",
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_PASS'
                )]) {
                    sh '''
                        echo $DOCKERHUB_PASS | docker login -u $DOCKERHUB_USER --password-stdin
                    '''
                    script { env.DOCKERHUB_USER = "${DOCKERHUB_USER}" }
                }
            }
        }

        stage('Build & Tag Images') {
            steps {
                script {
                    env.BACKEND_TAG_DH  = "${DOCKERHUB_USER}/three-tier-app-backend:${IMAGE_TAG}"
                    env.FRONTEND_TAG_DH = "${DOCKERHUB_USER}/three-tier-app-frontend:${IMAGE_TAG}"
                }
                sh '''
                    docker build -t ${BACKEND_TAG_DH} ./backend
                    docker build -t ${FRONTEND_TAG_DH} ./frontend
                '''
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                sh '''
                    docker push ${BACKEND_TAG_DH}
                    docker push ${FRONTEND_TAG_DH}
                '''
            }
        }

        stage('Prepare .env for Compose') {
            steps {
                writeFile file: '.env', text: """BACKEND_IMAGE=${BACKEND_TAG_DH}
FRONTEND_IMAGE=${FRONTEND_TAG_DH}
"""
            }
        }

        // Optional: gate stg/prod
        stage('Approval (Stg/Prod Only)') {
            when { anyOf { branch 'stg'; branch 'prod' } }
            steps {
                input message: "Deploy to ${BRANCH_NAME}?", ok: "Deploy"
            }
        }

        stage('Deploy Environment') {
            steps {
                sh """
                    docker-compose -f docker-compose.yml --env-file .env down
                    docker-compose -f docker-compose.yml --env-file .env pull
                    docker-compose -f docker-compose.yml --env-file .env up -d --remove-orphans
                """
            }
        }

        stage('Cleanup Local Images') {
            steps {
                sh 'docker rmi ${BACKEND_TAG_DH} ${FRONTEND_TAG_DH} || true'
            }
        }
    }

    post {
        success { echo "✅ ${BRANCH_NAME} environment deployed successfully! Tag: ${IMAGE_TAG}" }
        failure { echo "❌ Deployment failed for ${BRANCH_NAME}. Check logs." }
    }
}
