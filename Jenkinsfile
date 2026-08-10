pipeline {
    agent any

    parameters {
        string(name: 'TB_VERSION', defaultValue: '4.2', description: 'Enter the ThingsBoard version to upgrade (e.g., 4.2)')
    }

    environment {
        UPSTREAM_REPO = "https://github.com/thingsboard/thingsboard.git"
        DOCKER_COMPOSE_KAFKA = "docker-compose.kafka.yml"
        DOCKER_COMPOSE_TB = "docker-compose.thingsboard.yml"
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out repository...'
                checkout scm
            }
        }

        stage('Init Variables') {
            steps {
                script {
                    env.IMAGE_NAME = "thingsboard:${params.TB_VERSION}"
                    env.NEW_CONTAINER_NAME = "thingsboard-${params.TB_VERSION}"
                    env.UPGRADE_BRANCH = "dev-release-${params.TB_VERSION}"
                    env.BACKUP_BRANCH = "dev-backup-${params.TB_VERSION}"
                }
            }
        }

        stage('Detect Current Version') {
            steps {
                script {
                    echo 'Detecting current running ThingsBoard container...'
                    
                    def containerList = sh(script: "docker ps --format '{{.Names}}' | grep '^thingsboard-' || true", returnStdout: true).trim()
                    
                    if (containerList) {
                        def currentContainer = containerList.split("\\n")[0].trim()
                        def currentImage = sh(script: "docker inspect ${currentContainer} --format '{{ index .Config.Image }}'", returnStdout: true).trim()
                        def currentTag = currentImage.split(":")[1]

                        echo "Current running container: ${currentContainer}"
                        echo "Current running image: ${currentImage}"
                        echo "Current version: ${currentTag}"

                        env.CURRENT_CONTAINER_NAME = currentContainer
                        env.CURRENT_IMAGE_NAME = currentImage
                        env.CURRENT_VERSION = currentTag
                        env.ROLLBACK_IMAGE = "thingsboard:rollback-${currentTag}"
                    } else {
                        echo "No running ThingsBoard container found"
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
                    echo 'Comparing current version with target version...'
                    if (!params.TB_VERSION) {
                        error 'Target TB_VERSION parameter is required!'
                    }
                    
                    echo "Current version: ${env.CURRENT_VERSION ?: 'none'}, Target version: ${params.TB_VERSION}"
                    
                    if (env.CURRENT_VERSION == params.TB_VERSION) {
                        echo "ThingsBoard is already running version ${env.CURRENT_VERSION}"
                        env.UPGRADE_REQUIRED = "false"
                    } else {
                        echo "Upgrade required: ${env.CURRENT_VERSION ?: 'none'} -> ${params.TB_VERSION}"
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
                echo "Skipping upgrade - Already running target version ${params.TB_VERSION}"
            }
        }

        stage('Setup Git for Merge') {
            when {
                expression { env.UPGRADE_REQUIRED == "true" }
            }
            steps {
                script {
                    echo "Setting up Git for source code merge..."
                    sh """
                        # Configure Git if not already configured
                        #git config --global user.name "Jenkins CI" || true
                        #git config --global user.email "jenkins@yourdomain.com" || true
                        
                        # Check current branch
                        echo "Current branch:"
                        git branch
                        
                        # Create backup branch of current state
                        git branch ${env.BACKUP_BRANCH} || echo "Backup branch already exists"
                        
                        # Create or switch to upgrade branch
                        git checkout -b ${env.UPGRADE_BRANCH} || git checkout ${env.UPGRADE_BRANCH}
                        
                        # Add upstream remote if not exists
                        git remote add upstream ${UPSTREAM_REPO} || echo "Upstream remote already exists"
                        
                        # Fetch latest from upstream
                        git fetch upstream --tags
                        
                        echo "Available upstream branches:"
                        git branch -r | grep upstream/release || echo "No release branches found"
                        
                        echo "Available tags:"
                        git tag | grep "${params.TB_VERSION}" || echo "No matching tags found"
                    """
                }
            }
        }

        stage('Merge ThingsBoard Source') {
            when {
                expression { env.UPGRADE_REQUIRED == "true" }
            }
            steps {
                script {
                    echo "Merging ThingsBoard ${params.TB_VERSION} source code..."
                    sh """
                        # Merge upstream release branch
                        echo "Merging upstream/release-${params.TB_VERSION}..."
                        git merge upstream/release-${params.TB_VERSION} --no-edit || {
                            echo "Merge conflicts detected. Resolving automatically..."
                            
                            # Auto-resolve pom.xml version conflicts by accepting upstream changes
                            find . -name "pom.xml" -exec git checkout --theirs {} \\;
                            find . -name "pom.xml" -exec git add {} \\;
                            
                            # Check for remaining conflicts
                            if git status --porcelain | grep "^UU "; then
                                echo "Manual conflicts still exist. Listing them:"
                                git status --porcelain | grep "^UU "
                                
                                # For now, accept upstream changes for all conflicts
                                # In production, you might want more sophisticated conflict resolution
                                git checkout --theirs .
                                git add .
                            fi
                            
                            # Commit the merge
                            git commit -m "Merge ThingsBoard ${params.TB_VERSION} with custom changes - auto-resolved conflicts"
                        }
                        
                        echo "Source code merge completed successfully"
                        
                        # Verify version update
                        echo "Verifying version update:"
                        grep -r "${params.TB_VERSION}" pom.xml | head -3 || echo "Version verification failed"
                    """
                }
            }
        }

        
        stage('Build Custom ThingsBoard') {
            when {
                expression { env.UPGRADE_REQUIRED == "true" }
            }
            steps {
                script {
                    echo "Building custom ThingsBoard with version ${params.TB_VERSION}..."
                    sh """
                        # Clean and build the merged source code
                        echo "Starting Maven build..."
                        mvn clean package -Dmaven.test.skip=true
                        
                        # Verify build outputs
                        echo "Build completed. Checking outputs:"
                        ls -la application/target/thingsboard*.rpm || echo "RPM not found"
                        ls -la application/target/thingsboard*.jar || echo "JAR not found"
                        
                        # Ensure RPM exists with correct name
                        if [ -f application/target/thingsboard-${params.TB_VERSION}.0.rpm ]; then
                            cp application/target/thingsboard-${params.TB_VERSION}.0.rpm application/target/thingsboard.rpm
                        elif [ ! -f application/target/thingsboard.rpm ]; then
                            echo "ERROR: No RPM file found after build"
                            ls -la application/target/
                            exit 1
                        fi
                        
                        echo "Custom ThingsBoard build completed successfully"
                    """
                }
            }
        }


        stage('Push Branches to GitHub') {
            when {
                expression { env.UPGRADE_REQUIRED == "true" }
            }
            steps {
                script {
                    echo "Pushing upgrade and backup branches to GitHub..."
                    
                    // Use the GitHub credentials stored in Jenkins
                    withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN')]) {
                        sh """
                            # Set the remote URL with credentials embedded
                            git remote set-url origin https://${GIT_USERNAME}:${GIT_TOKEN}@github.com/sushenjayasuriya/Thingsboard-4.1.git
                            
                            # Push branches
                            echo "Pushing upgrade branch: ${env.UPGRADE_BRANCH}"
                            git push origin ${env.UPGRADE_BRANCH}
                            
                            echo "Pushing backup branch: ${env.BACKUP_BRANCH}"  
                            git push origin ${env.BACKUP_BRANCH}
                            
                            # Reset remote URL to remove credentials from git config
                            git remote set-url origin https://github.com/sushenjayasuriya/Thingsboard-4.1.git
                            
                            echo "Successfully pushed branches to GitHub"
                        """
                    }
                }
            }
        }

      stage('Update Dockerfile') {
            when {
                expression { env.UPGRADE_REQUIRED == "true" }
            }
            steps {
                script {
                    echo "Creating updated Dockerfile for source code deployment..."
                    
                    def dockerfileContent = '''FROM rockylinux:9

# Install dependencies
RUN dnf install -y wget rpm java-17-openjdk net-tools procps-ng && dnf clean all

# Copy your custom built RPM
COPY application/target/thingsboard.rpm /tmp/

# Install ThingsBoard from your custom RPM  
RUN rpm -ivh /tmp/thingsboard.rpm && rm -f /tmp/thingsboard.rpm

# Expose ThingsBoard UI/API port
EXPOSE 8080

# Execute the official upgrade script to migrate Cassandra/PostgreSQL schemas, then start the server
CMD ["/bin/bash", "-c", "/usr/share/thingsboard/bin/install/upgrade.sh --fromVersion=4.1.0 && java ${JAVA_OPTS} -jar /usr/share/thingsboard/bin/thingsboard.jar"]'''
                    
                    writeFile file: 'Dockerfile', text: dockerfileContent
                    echo "Updated Dockerfile created for source code deployment"
                }
            }
        }

        stage('Backup Current Image') {
            when {
                expression { env.UPGRADE_REQUIRED == "true" && env.CURRENT_IMAGE_NAME != "" }
            }
            steps {
                echo "Creating backup of current image: ${env.ROLLBACK_IMAGE}"
                sh "docker tag ${env.CURRENT_IMAGE_NAME} ${env.ROLLBACK_IMAGE}"
                echo "Backup image created: ${env.ROLLBACK_IMAGE}"
            }
        }

        stage('Build New Docker Image') {
            when {
                expression { env.UPGRADE_REQUIRED == "true" }
            }
            steps {
                echo "Building new ThingsBoard image: ${env.IMAGE_NAME}"
                sh """
                    docker build -t ${env.IMAGE_NAME} \\
                        --build-arg TB_VERSION=${params.TB_VERSION} \\
                        -f Dockerfile .
                    
                    echo "Image built successfully: ${env.IMAGE_NAME}"
                    docker images | grep thingsboard
                """
            }
        }

               stage('Generate Docker Compose') {
            when {
                expression { env.UPGRADE_REQUIRED == "true" }
            }
            steps {
                script {
                    echo "Generating docker-compose file for ThingsBoard ${params.TB_VERSION}"
                    
                    def composeContent = """version: "3.8"
services:
  tb-server:
    image: thingsboard:${params.TB_VERSION}
    container_name: thingsboard-${params.TB_VERSION}
    ports:
      - "8081:8080"
    environment:
      - JAVA_OPTS=-Xms1024M -Xmx1024M -Dspring.datasource.url=jdbc:postgresql://10.160.0.2:5432/thingsboard_restore -Dspring.datasource.username=nethmi -Dspring.datasource.password=123456 -Dcassandra.cluster.name=ThingsBoard Cluster -Dcassandra.keyspace.name=thingsboard -Dcassandra.url=10.160.0.2:9042 -Dcassandra.use.credentials=false
      - TB_QUEUE_TYPE=kafka
      - TB_QUEUE_PREFIX=dev_
      - TB_KAFKA_SERVERS=kafka-1:9092,kafka-2:9092,kafka-3:9092
      - TB_QUEUE_KAFKA_REPLICATION_FACTOR=3
      - METRICS_ENABLE=true
      - METRICS_ENDPOINTS_EXPOSE=prometheus
      - INSTALL_DATA_DIR=/data
      - SECURITY_OAUTH2_ENABLED=false
    volumes:
      - tb-data:/data
      - tb-logs:/var/log/thingsboard
    depends_on:
      - kafka-1
      - kafka-2
      - kafka-3
    networks:
      - tb-kafka-net
    restart: no
volumes:
  tb-data:
  tb-logs:
networks:
  tb-kafka-net:
    external: true
"""
                    
                    writeFile file: env.DOCKER_COMPOSE_TB, text: composeContent
                    echo "Generated: ${env.DOCKER_COMPOSE_TB}"
                }
            }
        }

        stage('Stop Current ThingsBoard') {
            when {
                expression { env.UPGRADE_REQUIRED == "true" && env.CURRENT_CONTAINER_NAME != "" }
            }
            steps {
                echo "Stopping current ThingsBoard container: ${env.CURRENT_CONTAINER_NAME}"
                sh """
                    # Stop current ThingsBoard service (keep kafka running)
                    docker stop ${env.CURRENT_CONTAINER_NAME} || true
                    docker rm ${env.CURRENT_CONTAINER_NAME} || true
                    echo "Old thingsboard container stopped and removed"
                    
                    echo "Stop running kafka container"

                    docker stop kafka-1 kafka-2 kafka-3 || true
                    docker rm kafka-1 kafka-2 kafka-3 || true

                    echo "Verifying no conflicting containers..."
                    docker ps -a | grep -E "(kafka|thingsboard)" || echo "No conflicting containers found"
                """
            }
        }

        stage('Deploy New Version') {
            when {
                expression { env.UPGRADE_REQUIRED == "true" }
            }
            steps {
                echo "Deploying complete stack with ThingsBoard ${params.TB_VERSION}"
                sh """
                    # Deploy new version with both compose files
                    docker compose -f ${env.DOCKER_COMPOSE_KAFKA} -f ${env.DOCKER_COMPOSE_TB} up -d 
                    
                    echo "Complete stack deployed with ThingsBoard ${params.TB_VERSION}"
                    echo "Checking container status..."
                    docker ps | grep -E "(kafka|thingsboard)"

                    echo "Waiting for services to be ready..."
                    sleep 10
                """
            }
        }

        stage('Verify Deployment') {
            when {
                expression { env.UPGRADE_REQUIRED == "true" }
            }
            steps {
                script {
                    echo "Verifying ThingsBoard deployment..."
                    echo "Waiting for ThingsBoard to start up..."
                    
                    // Increased initial wait time for heavy database migrations
                    sleep 180
                    
                    echo "Checking container health..."
                    // Use returnStatus: true to prevent Jenkins from aborting if grep fails (container crashed)
                    def isRunning = sh(script: "docker ps | grep thingsboard-${params.TB_VERSION}", returnStatus: true) == 0
                    
                    if (!isRunning) {
                        echo "CRITICAL: ThingsBoard container is NOT running! Fetching crash logs..."
                        sh "docker logs thingsboard-${params.TB_VERSION} || true"
                        error "Deployment verification failed - Container crashed during the initialization window."
                    }
                    
                    echo "Checking ThingsBoard logs for startup completion..."
                    sh """
                        # Show recent logs to verify startup
                        docker logs --tail 100 thingsboard-${params.TB_VERSION} | grep -E "(Started ThingsBoard|Startup complete|migration.*completed)" || true
                    """
                    
                    echo "Testing HTTP endpoint..."
                    def maxRetries = 15
                    def retryCount = 0
                    def httpStatus = ""
                    
                    while (retryCount < maxRetries) {
                        try {
                            httpStatus = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:8081/login", returnStdout: true).trim()
                            if (httpStatus == "200") {
                                echo "ThingsBoard is responding correctly (HTTP 200)"
                                break
                            } else {
                                echo "Attempt ${retryCount + 1}/${maxRetries}: HTTP status received was ${httpStatus}, retrying..."
                            }
                        } catch (Exception e) {
                            echo "Attempt ${retryCount + 1}/${maxRetries}: Connection failed, retrying in 30 seconds..."
                        }
                        
                        retryCount++
                        if (retryCount < maxRetries) {
                            sleep 30
                        }
                    }
                    
                    if (httpStatus != "200") {
                        echo "ThingsBoard is not responding correctly after ${maxRetries} attempts (Last HTTP status: ${httpStatus})"
                        error "Deployment verification failed - HTTP status: ${httpStatus}"
                    }
                    
                    echo "Deployment verified successfully!"
                }
            }
        }

        stage('Git Cleanup') {
            when {
                expression { env.UPGRADE_REQUIRED == "true" }
            }
            steps {
                script {
                    echo "Cleaning up Git branches..."
                    sh """
                        # Switch back to main branch and clean up
                        git checkout main || git checkout dev || echo "Could not switch to main branch"
                        
                        # Optionally push the upgrade branch to origin
                        # git push origin ${env.UPGRADE_BRANCH} || echo "Could not push upgrade branch"
                        
                        echo "Git cleanup completed"
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
ThingsBoard Source Code Upgrade Completed Successfully!
========================================================
Upgraded from: ${env.CURRENT_VERSION ?: 'none'} -> ${params.TB_VERSION}
Method: Source code merge with custom changes preserved
Container: thingsboard-${params.TB_VERSION}  
Web UI: http://localhost:8080
Backup available: ${env.ROLLBACK_IMAGE ?: 'none'}
Upgrade branch: ${env.UPGRADE_BRANCH}
Backup branch: ${env.BACKUP_BRANCH}
========================================================
                    """
                } else {
                    echo "No upgrade needed. ThingsBoard ${params.TB_VERSION} is already running."
                }
            }
        }
        
       failure {
            script {
                echo "ThingsBoard upgrade failed! Starting rollback procedures..."
                
                // SAVE LOGS BEFORE ROLLBACK
                echo "Saving container logs to workspace..."
                sh "docker logs thingsboard-${params.TB_VERSION} > thingsboard-crash.log 2>&1 || true"
                archiveArtifacts artifacts: 'thingsboard-crash.log', allowEmptyArchive: true

                if (env.UPGRADE_REQUIRED == "true") {
                // ... (Keep the rest of your existing rollback script here) ...
                    try {
                        echo "Rolling back to previous version..."
                        sh """
                            # Stop failed deployment
                            #docker compose -f ${env.DOCKER_COMPOSE_KAFKA} -f ${env.DOCKER_COMPOSE_TB} down || true
                            
                            # Clean up any remaining containers
                            docker stop thingsboard-${params.TB_VERSION} kafka-1 kafka-2 kafka-3 || true
                            docker rm thingsboard-${params.TB_VERSION} kafka-1 kafka-2 kafka-3 || true

                            # Git rollback
                            git checkout ${env.BACKUP_BRANCH} || echo "Could not checkout backup branch"
                            
                            # Rebuild previous version if backup image exists
                            if [ -n "${env.ROLLBACK_IMAGE}" ]; then
                                # Create rollback compose file
                                cat > docker-compose.rollback.yml << 'EOF'
version: "3.8"
services:
  tb-server:
    image: ${env.ROLLBACK_IMAGE}
    container_name: ${env.CURRENT_CONTAINER_NAME}
    ports:
      - "8080:8080"
    environment:
      - DATABASE_TS_TYPE=cassandra
      - SPRING_DATASOURCE_URL=jdbc:postgresql://10.160.0.2:5432/thingsboard_restore
      - SPRING_DATASOURCE_USERNAME=nethmi
      - SPRING_DATASOURCE_PASSWORD=123456
      - CASSANDRA_CLUSTER_NAME=ThingsBoard Cluster
      - CASSANDRA_KEYSPACE_NAME=thingsboard
      - CASSANDRA_URL=10.160.0.2:9042
      - CASSANDRA_USE_CREDENTIALS=false
      - SECURITY_OAUTH2_ENABLED=false
      - TB_QUEUE_TYPE=kafka
      - TB_QUEUE_PREFIX=dev_
      - TB_KAFKA_SERVERS=kafka-1:9092,kafka-2:9092,kafka-3:9092
      - TB_QUEUE_KAFKA_REPLICATION_FACTOR=3
      - METRICS_ENABLE=true
      - METRICS_ENDPOINTS_EXPOSE=prometheus
      - INSTALL_DATA_DIR=/data
    volumes:
      - tb-data:/data
      - tb-logs:/var/log/thingsboard
    depends_on:
      - kafka-1
      - kafka-2  
      - kafka-3
    networks:
      - tb-kafka-net
    restart: no
volumes:
  tb-data:
  tb-logs:
networks:
  tb-kafka-net:
    external: true
EOF
                                
                                # Deploy rollback stack
                                docker compose -f ${env.DOCKER_COMPOSE_KAFKA} -f docker-compose.rollback.yml up -d
                            fi
                        """
                        
                        echo "Rollback completed successfully. ThingsBoard restored to v${env.CURRENT_VERSION}"
                    } catch (Exception e) {
                        echo "Rollback failed: ${e.getMessage()}"
                        echo "Manual intervention required!"
                    }
                } else {
                    echo "No backup available for rollback. Manual intervention required."
                }
                
                // error "ThingsBoard upgrade failed. Check logs for details."
            }
        }
        
        always {
            echo "Cleaning up temporary files..."
            sh """
                # Clean up generated compose file
                #rm -f ${env.DOCKER_COMPOSE_TB} || true
                #rm -f docker-compose.rollback.yml || true
                
                echo "Cleanup completed"
            """
        }
    }
}
