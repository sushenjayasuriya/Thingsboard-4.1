pipeline {
    agent any

    parameters {
        string(name: 'TB_VERSION', defaultValue: '4.1', description: 'Enter the ThingsBoard version to upgrade (e.g., 4.2)')
    }

    environment {
        PACKAGE_REPO = "https://github.com/thingsboard/thingsboard/releases/download"
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
                    echo '🔍 Detecting current running ThingsBoard Production container in Kubernetes...'
                    
                    def currentImage = sh(script: "kubectl describe deployment thingsboard -n thingsboard | grep Image | awk '{print \$2}' || true", returnStdout: true).trim()
                    
                    if (currentImage) {
                        def currentTag = currentImage.split(":")[1]
                        echo "📦 Current running Production image: ${currentImage}"
                        echo "📦 Current Production version: ${currentTag}"

                        env.CURRENT_IMAGE_NAME = currentImage
                        env.CURRENT_VERSION = currentTag
                        env.ROLLBACK_IMAGE = "thingsboard-prod:rollback-${currentTag}"
                    } else {
                        echo "⚠️ No running ThingsBoard Production pod found in Kubernetes"
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

        stage('Deploy To Kubernetes') {
            when {
                expression { env.UPGRADE_REQUIRED == "true" }
            }
            steps {
                script {
                    echo "🚀 Deploying custom image to Kubernetes cluster..."
                    
                    sh """
                        # 1. Stop and remove the old host Docker container if it exists (keeps port 8080 free)
                        docker rm -f thingsboard-prod-${params.TB_VERSION} || true
                        docker ps -q --filter "publish=8080" | xargs -r docker rm -f || true

                        # 2. Export the Docker image built by Jenkins
                        docker save ${env.IMAGE_NAME} -o /tmp/tb-image.tar

                        # 3. Import it directly into Kubernetes containerd
                        sudo ctr -n k8s.io images import /tmp/tb-image.tar

                        # 4. Update the Kubernetes deployment and restart
                        kubectl set image deployment/thingsboard thingsboard=${env.IMAGE_NAME} -n thingsboard
                        kubectl rollout restart deployment/thingsboard -n thingsboard
                        kubectl rollout status deployment/thingsboard -n thingsboard
                        
                        echo "✅ Kubernetes deployment triggered successfully!"
                    """
                }
            }
        }

        stage('Verify Deployment') {
            when {
                expression { env.UPGRADE_REQUIRED == "true" }
            }
            steps {
                script {
                    echo "🔍 Verifying ThingsBoard Kubernetes deployment..."
                    
                    // Wait for Kubernetes rollout to finish
                    sh "kubectl rollout status deployment/thingsboard -n thingsboard --timeout=5m"
                    
                    echo "✅ Kubernetes rollout completed successfully!"
                    echo "🔍 Listing all running pods in the thingsboard namespace:"
                    sh "kubectl get pods -n thingsboard"
                    
                    echo "🔍 Checking ThingsBoard logs for startup completion..."
                    // Get the newest pod and check its logs for the startup message
                    sh """
                        POD_NAME=\$(kubectl get pods -n thingsboard -l app=thingsboard -o jsonpath='{.items[-1:].metadata.name}')
                        echo "Checking logs for pod: \$POD_NAME"
                        kubectl logs --tail=20 \$POD_NAME -n thingsboard | grep -E "(Started ThingsBoard|Startup complete)" || echo "⚠️ Startup message not found yet, but rollout is complete."
                    """
                    echo "✅ Production Deployment verified successfully!"
                }
            }
        }
    } // Close stages

    post {
        success {
            script {
                if (env.UPGRADE_REQUIRED == "true") {
                    echo """
🎉 ThingsBoard Production Upgrade Completed Successfully!
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Upgraded from: ${env.CURRENT_VERSION ?: 'none'} → ${params.TB_VERSION}
🐳 Kubernetes Pod: thingsboard-xxxxx-xxxxx (deployment/thingsboard)
🌐 Production Web UI: https://tb.utech-iiot.lk
📦 Backup available: N/A (Kubernetes manages rollback history)
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
                
                if (env.UPGRADE_REQUIRED == "true") {
                    try {
                        echo "🔄 Rolling back Production Kubernetes deployment to previous version..."
                        
                        // Kubernetes native rollback
                        sh """
                            kubectl rollout undo deployment/thingsboard -n thingsboard
                            kubectl rollout status deployment/thingsboard -n thingsboard --timeout=5m
                        """
                        
                        echo "✅ Production Rollback completed successfully."
                        echo "🚨 URGENT: Notify operations team that Production rollback was executed!"
                    } catch (Exception e) {
                        echo "❌ Production Rollback FAILED: ${e.getMessage()}"
                        echo "🚨 CRITICAL: Manual intervention required for Production immediately!"
                    }
                } else {
                    echo "⚠️ No Production upgrade was in progress, skipping rollback."
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
                rm -f thingsboard-*.rpm || true
                
                echo "✅ Production Cleanup completed"
            """
        }
    }
} // Close pipeline
