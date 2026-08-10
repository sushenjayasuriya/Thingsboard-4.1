pipeline {
    agent any

    parameters {
        string(name: 'TB_VERSION', defaultValue: '4.1', description: 'Enter the ThingsBoard version to upgrade (e.g., 4.2)')
    }

    environment {
        PACKAGE_REPO = "https://github.com/thingsboard/thingsboard/releases/download"
        // DOCKER_COMPOSE_KAFKA = "docker-compose.kafka.yml"
        DOCKER_COMPOSE_TB = "docker-compose.prod.yml"
    }

    stages {
        stage('Checkout') {
            steps {
                echo '📥 Checking out Production repository...'
                checkout scm
            }
        }

        stage('Init Variables') {
            steps {
                script {
                    env.IMAGE_NAME = "thingsboard-prod:${params.TB_VERSION}"
                    env.NEW_CONTAINER_NAME = "thingsboard-prod-${params.TB_VERSION}"
                }
            }
        }

        stage('Detect Current Installed Version') {
            steps {
                script {
                    echo '🔍 Detecting current running ThingsBoard Production container...'
                    
                    def containerList = sh(script: "docker ps --format '{{.Names}}' | grep '^thingsboard-prod-' || true", returnStdout: true).trim()
                    
                    if (containerList) {
                        def currentContainer = containerList.split("\\n")[0].trim()
                        def currentImage = sh(script: "docker inspect ${currentContainer} --format '{{ index .Config.Image }}'", returnStdout: true).trim()
                        def currentTag = currentImage.split(":")[1]

                        echo "📦 Current running Production container: ${currentContainer}"
                        echo "📦 Current running Production image: ${currentImage}"
                        echo "📦 Current Production version: ${currentTag}"

                        env.CURRENT_CONTAINER_NAME = currentContainer
                        env.CURRENT_IMAGE_NAME = currentImage
                        env.CURRENT_VERSION = currentTag
                        env.ROLLBACK_IMAGE = "thingsboard-prod:rollback-${currentTag}"
                    } else {
                        echo "⚠️ No running ThingsBoard Production container found"
                        env.CURRENT_CONTAINER_NAME = ""
                        env.CURRENT_VERSION = "none"
                        env.CURRENT_IMAGE_NAME = ""
                        env.ROLLBACK_IMAGE = ""
                    }
                }
            }
        }

        stage('Compare Versions') {
            steps {
                script {
                    echo '🔍 Comparing current Production version with target version...'
                    if (!params.TB_VERSION) {
                        error '❌ Target TB_VERSION parameter is required!'
                    }
                    
                    echo "📦 Current Production version: ${env.CURRENT_VERSION ?: 'none'}, Target version: ${params.TB_VERSION}"
                    
                    if (env.CURRENT_VERSION == params.TB_VERSION) {
                        echo "✅ ThingsBoard Production is already running version ${env.CURRENT_VERSION}"
                        env.UPGRADE_REQUIRED = "false"
                    } else {
                        echo "⬆️ Production Upgrade required: ${env.CURRENT_VERSION ?: 'none'} ➜ ${params.TB_VERSION}"
                        env.UPGRADE_REQUIRED = "true"
                    }
                }
            }
        }

        stage('Skip Upgrade') {
            when {
                expression { env.UPGRADE_REQUIRED == "false" }
            }
            steps {
                echo "✅ Skipping Production upgrade — Already running target version ${params.TB_VERSION}"
            }
        }

        stage('Download RPM') {
            when {
                expression { env.UPGRADE_REQUIRED == "true" }
            }
            steps {
                script {
                    echo "📥 Downloading ThingsBoard RPM package for Production..."
                    def rpmUrl = "${PACKAGE_REPO}/v${params.TB_VERSION}/thingsboard-${params.TB_VERSION}.rpm"
                    echo "📥 Downloading RPM from: ${rpmUrl}"
                    
                    sh """
                        # Download the RPM package
                        curl -L -o thingsboard-${params.TB_VERSION}.rpm ${rpmUrl}
                        ls -lh thingsboard-*.rpm
                    
                        # Prepare application directory structure
                        mkdir -p application/target
                        cp thingsboard-${params.TB_VERSION}.rpm application/target/thingsboard.rpm
                        
                        echo "✅ RPM downloaded and copied to application/target/"
                    """
                }
            }
        }

        stage('Backup Current Image') {
            when {
                expression { env.UPGRADE_REQUIRED == "true" && env.CURRENT_IMAGE_NAME != "" }
            }
            steps {
                echo "📦 Creating backup of current Production image: ${env.ROLLBACK_IMAGE}"
                sh "docker tag ${env.CURRENT_IMAGE_NAME} ${env.ROLLBACK_IMAGE}"
                echo "✅ Production backup image created: ${env.ROLLBACK_IMAGE}"
            }
        }

        stage('Build New Docker Image') {
            when {
                expression { env.UPGRADE_REQUIRED == "true" }
            }
            steps {
                echo "🔧 Building new ThingsBoard Production image: ${env.IMAGE_NAME}"
                sh """
                    docker build -t ${env.IMAGE_NAME} \
                        --build-arg TB_VERSION=${params.TB_VERSION} \
                        -f Dockerfile.prod .
                    
                    echo "✅ Production image built successfully: ${env.IMAGE_NAME}"
                    docker images | grep thingsboard-prod
                """
            }
        }

        stage('Generate Docker Compose') {
            when {
                expression { env.UPGRADE_REQUIRED == "true" }
            }
            steps {
                script {
                    echo "📝 Generating docker-compose file for ThingsBoard Production ${params.TB_VERSION}"
                    
                    def composeContent = """version: "3.8"
services:
  tb-server:
    image: thingsboard-prod:${params.TB_VERSION}
    container_name: thingsboard-prod-${params.TB_VERSION}
    ports:
      - "8080:8080"
    environment:
      - JAVA_OPTS=-Xms2048M -Xmx2048M
      - DATABASE_TS_TYPE=cassandra
      - SPRING_DATASOURCE_URL=jdbc:postgresql://10.160.0.3:5432/thingsboard_prod
      - SPRING_DATASOURCE_USERNAME=nethmi
      - SPRING_DATASOURCE_PASSWORD=123456
      - CASSANDRA_CLUSTER_NAME=ThingsBoard Cluster
      - CASSANDRA_KEYSPACE_NAME=thingsboard_prod
      - CASSANDRA_URL=10.160.0.3:9042
      - CASSANDRA_USE_CREDENTIALS=false
      - SECURITY_OAUTH2_ENABLED=true
      - TB_QUEUE_TYPE=kafka
      - TB_QUEUE_PREFIX=prod_
      - TB_KAFKA_SERVERS=kafka:9092
      - METRICS_ENABLE=true
      - METRICS_ENDPOINTS_EXPOSE=prometheus
    networks:
      - tb-kafka-net
    restart: no
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  tb-kafka-net:
    external: true
"""
                    
                    writeFile file: env.DOCKER_COMPOSE_TB, text: composeContent
                    echo "✅ Generated Production compose file: ${env.DOCKER_COMPOSE_TB}"
                }
            }
        }

        stage('Stop Current ThingsBoard') {
            steps {
                echo "🛑 Forcibly clearing old ThingsBoard Production containers and proxies..."
                sh """
                    # Force stop and remove containers cleanly
                    docker rm -f thingsboard-prod-4.1 thingsboard-prod-4.0 || true
                    
                    # Kill any lingering docker-proxy or process holding port 8080
                    sudo fuser -k 8080/tcp || true
                    
                    # Give the OS kernel a moment to release port 8080 bindings
                    sleep 3
                    
                    echo "✅ Port 8080 fully released and cleared"
                """
            }
        }

        stage('Deploy New Version') {
            when {
                expression { env.UPGRADE_REQUIRED == "true" }
            }
            steps {
                echo "🚀 Deploying complete Production stack with ThingsBoard ${params.TB_VERSION}"
                sh """
                    # Deploy new Production version with both compose files
                    docker compose -f ${env.DOCKER_COMPOSE_TB} up -d
                    
                    echo "✅ Complete Production stack deployed with ThingsBoard ${params.TB_VERSION}"
                    echo "🔍 Checking Production container status..."
                    docker ps | grep -E "(thingsboard-prod)"

                    echo "🔍 Waiting for Production services to be ready..."
                    sleep 15

                """
            }
        }

        stage('Verify Deployment') {
            when {
                expression { env.UPGRADE_REQUIRED == "true" }
            }
            steps {
                script {
                    echo "🔍 Verifying ThingsBoard Production deployment..."
                    echo "⏳ Waiting for ThingsBoard Production to start up..."
                    
                    // Wait longer for production startup
                    sleep 90
                    
                    echo "🔍 Checking Production container health..."
                    sh "docker ps | grep thingsboard-prod-${params.TB_VERSION}"
                    
                    echo "🔍 Checking ThingsBoard Production logs for startup completion..."
                    sh """
                        # Show recent logs to verify startup
                        docker logs --tail 50 thingsboard-prod-${params.TB_VERSION} | grep -E "(Started ThingsBoard|Startup complete)" || true
                    """
                    
                    echo "🌐 Testing Production HTTP endpoint..."
                    // Test the Production web interface
                    def maxRetries = 8  // More retries for production
                    def retryCount = 0
                    def httpStatus = ""
                    
                    while (retryCount < maxRetries) {
                        try {
                            httpStatus = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:8080/login", returnStdout: true).trim()
                            if (httpStatus == "200") {
                                echo "✅ ThingsBoard Production is responding correctly (HTTP 200)"
                                break
                            }
                        } catch (Exception e) {
                            echo "⏳ Attempt ${retryCount + 1}/${maxRetries}: HTTP status ${httpStatus}, retrying in 30 seconds..."
                        }
                        
                        retryCount++
                        if (retryCount < maxRetries) {
                            sleep 30
                        }
                    }
                    
                    if (httpStatus != "200") {
                        echo "❌ ThingsBoard Production is not responding correctly after ${maxRetries} attempts (HTTP ${httpStatus})"
                        error "❌ Production Deployment verification failed — HTTP status: ${httpStatus}"
                    }
                    
                    echo "🎉 Production Deployment verified successfully!"
                    
                    // Additional Production checks
                    echo "🔒 Running basic Production health checks..."
                    sh """
                        # Check if container is healthy
                        docker inspect thingsboard-prod-${params.TB_VERSION} --format='{{.State.Health.Status}}' || echo "No health check defined"
                        
                        # Check memory usage
                        docker stats --no-stream --format "Memory: {{.MemUsage}}" thingsboard-prod-${params.TB_VERSION}
                        
                        echo "✅ Production health checks completed"
                    """
                }
            }
        }
    }

    post {
        success {
            script {
                if (env.UPGRADE_REQUIRED == "true") {
                    echo """
🎉 ThingsBoard Production Upgrade Completed Successfully!
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Upgraded from: ${env.CURRENT_VERSION ?: 'none'} → ${params.TB_VERSION}
🐳 Production Container: thingsboard-prod-${params.TB_VERSION}
🌐 Production Web UI: http://localhost:8080
📦 Backup available: ${env.ROLLBACK_IMAGE ?: 'none'}
🔒 Security: OAuth2 Enabled
📊 Monitoring: Enabled
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️  IMPORTANT: Monitor Production closely for the next few hours!
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                    """
                } else {
                    echo "✅ No Production upgrade needed. ThingsBoard Production ${params.TB_VERSION} is already running."
                }
            }
        }
        
        failure {
            script {
                echo "❌ ThingsBoard Production upgrade FAILED! Starting EMERGENCY rollback procedures..."
                
                if (env.UPGRADE_REQUIRED == "true" && env.ROLLBACK_IMAGE && env.CURRENT_CONTAINER_NAME) {
                    try {
                        echo "🔄 Rolling back Production to previous version..."

                        sh """
                            # Stop failed Production deployment
                            docker compose -f ${env.DOCKER_COMPOSE_TB} down || true
                            
                            # Clean up any remaining containers
                            docker stop thingsboard-prod-${params.TB_VERSION} || true
                            docker rm thingsboard-prod-${params.TB_VERSION} || true
                            
                            # Restore previous Production version with Docker Compose approach
                            # First, create a rollback compose file
                            cat > docker-compose.rollback.prod.yml << 'EOF'
version: "3.8"
services:
  tb-server:
    image: ${env.ROLLBACK_IMAGE}
    container_name: ${env.CURRENT_CONTAINER_NAME}
    ports:
      - "8080:8080"
    environment:
      - DATABASE_TS_TYPE=cassandra
      - SPRING_DATASOURCE_URL=jdbc:postgresql://10.160.0.3:5432/thingsboard_prod
      - SPRING_DATASOURCE_USERNAME=nethmi
      - SPRING_DATASOURCE_PASSWORD=123456
      - CASSANDRA_CLUSTER_NAME=ThingsBoard Cluster
      - CASSANDRA_KEYSPACE_NAME=thingsboard_prod
      - CASSANDRA_URL=10.160.0.3:9042
      - CASSANDRA_USE_CREDENTIALS=false
      - SECURITY_OAUTH2_ENABLED=true
      - TB_QUEUE_TYPE=kafka
      - TB_QUEUE_PREFIX=prod_
      - TB_KAFKA_SERVERS=kafka:9092
      - METRICS_ENABLE=true
      - METRICS_ENDPOINTS_EXPOSE=prometheus
    networks:
      - tb-kafka-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/login"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  tb-kafka-net:
    external: true
EOF
                            
                            # Deploy rollback stack for Production
                            docker compose -f docker-compose.rollback.prod.yml up -d
                            
                            # Wait and verify rollback
                            sleep 60
                            curl -f http://localhost:8080/login || echo "⚠️ Production rollback verification failed"
                        """

                        
                        echo "✅ Production Rollback completed successfully. ThingsBoard Production restored to v${env.CURRENT_VERSION}"
                        echo "🚨 URGENT: Notify operations team that Production rollback was executed!"
                    } catch (Exception e) {
                        echo "❌ Production Rollback FAILED: ${e.getMessage()}"
                        echo "🚨 CRITICAL: Manual intervention required for Production immediately!"
                    }
                } else {
                    echo "⚠️ No Production backup available for rollback. CRITICAL: Manual intervention required!"
                }
                
                error "❌ ThingsBoard Production upgrade failed. URGENT: Check logs and notify operations team!"
            }
        }
        
        unstable {
            echo "⚠️ ThingsBoard Production upgrade completed but may be unstable. Monitor very closely!"
        }
        
        always {
            echo "🧹 Cleaning up Production temporary files..."
            sh """
                # Clean up downloaded RPM files
                #rm -f thingsboard-*.rpm || true
                
                # Clean up generated compose file
                rm -f ${env.DOCKER_COMPOSE_TB} || true
                
                echo "✅ Production Cleanup completed"
            """
        }
    }
}
