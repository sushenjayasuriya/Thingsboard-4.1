pipeline {
    agent any

    parameters {
        string(name: 'TB_VERSION', defaultValue: '4.2', description: 'ThingsBoard version to deploy (e.g., 4.2)')
    }

    environment {
        // Name of the container and image
        CONTAINER_NAME = "thingsboard-${params.TB_VERSION}"
        IMAGE_NAME = "thingsboard:${params.TB_VERSION}"
        // Path to your existing docker-compose.yml
        COMPOSE_FILE = "docker-compose.yml"
    }

    stages {
        stage('Checkout') {
            steps {
                echo '📥 Checking out QA repository...'
                checkout scm
            }
        }

        stage('Detect Current Version') {
            steps {
                script {
                    echo '🔍 Checking for running ThingsBoard container...'
                    def running = sh(script: "docker ps --format '{{.Names}}' | grep '^thingsboard-' || true", returnStdout: true).trim()
                    if (running) {
                        env.CURRENT_VERSION = sh(script: "docker inspect ${running} --format '{{ index .Config.Image }}' | cut -d: -f2", returnStdout: true).trim()
                        echo "📦 Current running container: ${running} (version ${env.CURRENT_VERSION})"
                    } else {
                        env.CURRENT_VERSION = "none"
                        echo "⚠️ No running ThingsBoard container found"
                    }
                }
            }
        }

        stage('Compare Versions') {
            steps {
                script {
                    if (env.CURRENT_VERSION == params.TB_VERSION) {
                        echo "✅ ThingsBoard ${params.TB_VERSION} is already running. Skipping upgrade."
                        env.SKIP_UPGRADE = "true"
                    } else {
                        echo "⬆️ Upgrade required: ${env.CURRENT_VERSION} → ${params.TB_VERSION}"
                        env.SKIP_UPGRADE = "false"
                    }
                }
            }
        }

        stage('Download RPM') {
            when {
                expression { env.SKIP_UPGRADE == "false" }
            }
            steps {
                script {
                    echo "📥 Downloading ThingsBoard RPM for version ${params.TB_VERSION}..."
                    sh """
                        if [ ! -f thingsboard-${params.TB_VERSION}.rpm ]; then
                            curl -L -o thingsboard-${params.TB_VERSION}.rpm \
                                https://github.com/thingsboard/thingsboard/releases/download/v${params.TB_VERSION}/thingsboard-${params.TB_VERSION}.rpm
                        else
                            echo "RPM already exists, skipping download"
                        fi
                    """
                }
            }
        }

        stage('Build Docker Image') {
            when {
                expression { env.SKIP_UPGRADE == "false" }
            }
            steps {
                echo "🔧 Building Docker image: ${env.IMAGE_NAME}"
                sh """
                    docker build -t ${env.IMAGE_NAME} \
                        --build-arg TB_VERSION=${params.TB_VERSION} \
                        -f Dockerfile .
                """
            }
        }

        stage('Stop Current Containers') {
            when {
                expression { env.SKIP_UPGRADE == "false" }
            }
            steps {
                echo "🛑 Stopping old containers..."
                sh """
                    docker compose -f ${env.COMPOSE_FILE} down || true
                """
            }
        }

        stage('Start Dependencies (Postgres, Kafka, Zookeeper)') {
            when {
                expression { env.SKIP_UPGRADE == "false" }
            }
            steps {
                echo "📦 Starting PostgreSQL, Kafka, Zookeeper..."
                sh """
                    docker compose -f ${env.COMPOSE_FILE} up -d postgres zookeeper kafka
                    sleep 10
                """
            }
        }

        stage('Run Database Installer') {
            when {
                expression { env.SKIP_UPGRADE == "false" }
            }
            steps {
                echo "⚙️ Running ThingsBoard installer to initialise database..."
                sh """
                    docker run --rm --network thingsboard-qa_default \
                        -e DATABASE_TS_TYPE=sql \
                        -e SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/thingsboard \
                        -e SPRING_DATASOURCE_USERNAME=postgres \
                        -e SPRING_DATASOURCE_PASSWORD=postgres \
                        ${env.IMAGE_NAME} /usr/share/thingsboard/bin/install/install.sh
                """
            }
        }

        stage('Start ThingsBoard Service') {
            when {
                expression { env.SKIP_UPGRADE == "false" }
            }
            steps {
                echo "🚀 Starting ThingsBoard ${params.TB_VERSION}..."
                sh """
                    docker compose -f ${env.COMPOSE_FILE} up -d tb-server
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    echo "🔍 Verifying ThingsBoard is up..."
                    // Wait for startup
                    sleep 30

                    // Check container is running
                    def isRunning = sh(script: "docker ps --format '{{.Names}}' | grep '^${env.CONTAINER_NAME}$'", returnStatus: true) == 0
                    if (!isRunning) {
                        error "❌ Container ${env.CONTAINER_NAME} is not running."
                    }

                    // Check HTTP endpoint
                    def maxRetries = 12
                    def retryCount = 0
                    def httpStatus = ""
                    while (retryCount < maxRetries) {
                        try {
                            httpStatus = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:8080/login", returnStdout: true).trim()
                            if (httpStatus == "200") {
                                echo "✅ ThingsBoard responded with HTTP 200"
                                break
                            }
                        } catch (Exception e) {
                            echo "⏳ Attempt ${retryCount+1}/${maxRetries}: HTTP ${httpStatus}, retrying..."
                        }
                        retryCount++
                        if (retryCount < maxRetries) sleep 10
                    }

                    if (httpStatus != "200") {
                        error "❌ ThingsBoard did not return HTTP 200 after ${maxRetries} attempts."
                    }

                    echo "🎉 QA deployment verified successfully!"
                }
            }
        }
    }

    post {
        success {
            echo """
🎉 ThingsBoard QA ${params.TB_VERSION} is up and running!
📦 Container: ${env.CONTAINER_NAME}
🌐 URL: http://localhost:8080 (or via HAProxy domain)
            """
        }
        failure {
            echo "❌ QA deployment failed. Check logs above."
            script {
                // Optionally, you can add rollback logic here (but we keep it simple)
            }
        }
        always {
            echo "🧹 Cleanup (optional) – removing temp files..."
            sh """
                # Keep the RPM for reuse
                echo "RPM kept at thingsboard-${params.TB_VERSION}.rpm"
            """
        }
    }
}
