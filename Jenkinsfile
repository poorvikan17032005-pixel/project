pipeline {
    agent any

    environment {
        APP_NAME        = 'votesecure'
        IMAGE_NAME      = "${APP_NAME}:${BUILD_NUMBER}"
        IMAGE_LATEST    = "${APP_NAME}:latest"
        CONTAINER_NAME  = "${APP_NAME}_app"
        APP_PORT        = '5000'
        DOCKER_REGISTRY = ''   // e.g. 'docker.io/yourusername' — leave blank for local
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 20, unit: 'MINUTES')
        timestamps()
    }

    stages {

        // ── 1. CHECKOUT ──────────────────────────────────────────────────────
        stage('Checkout') {
            steps {
                echo '==> Cloning repository...'
                checkout scm
                sh 'echo "Branch: ${GIT_BRANCH}" && echo "Commit: ${GIT_COMMIT}"'
            }
        }

        // ── 2. INSTALL DEPENDENCIES ──────────────────────────────────────────
        stage('Install Dependencies') {
            steps {
                echo '==> Installing Python dependencies...'
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        // ── 3. RUN TESTS ─────────────────────────────────────────────────────
        stage('Run Tests') {
            steps {
                echo '==> Running unit tests...'
                sh '''
                    . venv/bin/activate
                    pytest tests/ -v --tb=short --junitxml=test-results.xml
                '''
            }
            post {
                always {
                    junit 'test-results.xml'
                }
            }
        }

        // ── 4. CODE QUALITY LINT ─────────────────────────────────────────────
        stage('Code Quality') {
            steps {
                echo '==> Running lint checks...'
                sh '''
                    . venv/bin/activate
                    pip install flake8 --quiet
                    flake8 app/ --max-line-length=120 --exclude=__pycache__ || true
                '''
            }
        }

        // ── 5. BUILD DOCKER IMAGE ─────────────────────────────────────────────
        stage('Build Docker Image') {
            steps {
                echo "==> Building Docker image: ${IMAGE_NAME}..."
                sh '''
                    docker build \
                        --target production \
                        -t ${IMAGE_NAME} \
                        -t ${IMAGE_LATEST} \
                        --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
                        --build-arg VCS_REF=${GIT_COMMIT} \
                        .
                '''
            }
        }

        // ── 6. TEST DOCKER IMAGE ──────────────────────────────────────────────
        stage('Test Docker Image') {
            steps {
                echo '==> Smoke testing Docker image...'
                sh '''
                    docker run -d \
                        --name ${APP_NAME}_test_${BUILD_NUMBER} \
                        -p 5099:5000 \
                        -e DATABASE_PATH=/tmp/test.db \
                        ${IMAGE_NAME}

                    sleep 8

                    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:5099/)
                    echo "HTTP Status: $STATUS"

                    docker stop ${APP_NAME}_test_${BUILD_NUMBER}
                    docker rm  ${APP_NAME}_test_${BUILD_NUMBER}

                    if [ "$STATUS" != "200" ]; then
                        echo "Smoke test FAILED — HTTP $STATUS"
                        exit 1
                    fi
                    echo "Smoke test PASSED"
                '''
            }
        }

        // ── 7. PUSH TO REGISTRY (optional) ───────────────────────────────────
        stage('Push to Registry') {
            when {
                allOf {
                    branch 'main'
                    expression { return env.DOCKER_REGISTRY != '' }
                }
            }
            steps {
                echo '==> Pushing image to Docker registry...'
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker tag ${IMAGE_NAME} ${DOCKER_REGISTRY}/${IMAGE_NAME}
                        docker tag ${IMAGE_LATEST} ${DOCKER_REGISTRY}/${IMAGE_LATEST}
                        docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}
                        docker push ${DOCKER_REGISTRY}/${IMAGE_LATEST}
                    '''
                }
            }
        }

        // ── 8. DEPLOY ─────────────────────────────────────────────────────────
        stage('Deploy') {
            when { branch 'main' }
            steps {
                echo '==> Deploying application...'
                sh '''
                    # Stop and remove old container if running
                    docker stop ${CONTAINER_NAME} 2>/dev/null || true
                    docker rm   ${CONTAINER_NAME} 2>/dev/null || true

                    # Start new container
                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        --restart unless-stopped \
                        -p ${APP_PORT}:5000 \
                        -v votesecure_data:/app/data \
                        -e FLASK_ENV=production \
                        -e DATABASE_PATH=/app/data/voting.db \
                        ${IMAGE_NAME}

                    echo "==> Waiting for app to start..."
                    sleep 10

                    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:${APP_PORT}/)
                    echo "Deployment health check: HTTP $STATUS"

                    if [ "$STATUS" != "200" ]; then
                        echo "Deployment health check FAILED"
                        exit 1
                    fi

                    echo "==> Deployment successful! App running on port ${APP_PORT}"
                '''
            }
        }

        // ── 9. CLEANUP ────────────────────────────────────────────────────────
        stage('Cleanup') {
            steps {
                echo '==> Cleaning up dangling images...'
                sh '''
                    docker image prune -f || true
                    rm -rf venv || true
                '''
            }
        }
    }

    post {
        success {
            echo """
            ============================================
             BUILD SUCCESS
             App     : ${APP_NAME}
             Version : ${BUILD_NUMBER}
             Branch  : ${GIT_BRANCH}
            ============================================
            """
        }
        failure {
            echo """
            ============================================
             BUILD FAILED — Check console output above
             App     : ${APP_NAME}
             Version : ${BUILD_NUMBER}
            ============================================
            """
            // Add email/Slack notification here if needed
            // mail to: 'team@example.com', subject: "Build Failed: ${APP_NAME} #${BUILD_NUMBER}"
        }
        always {
            cleanWs()
        }
    }
}
