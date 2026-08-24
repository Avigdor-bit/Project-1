pipeline {
    // ============================================================
    // 1. AGENT - Defines where pipeline runs
    // ============================================================
    agent any  // Runs on any available Jenkins agent/worker
    
    // ============================================================
    // 2. PARAMETERS - Build-time user inputs
    // ============================================================
    parameters {
        string(name: 'BRANCH', defaultValue: 'main', description: 'Git branch to build')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'production'], description: 'Deploy target')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip unit tests')
    }
    
    // ============================================================
    // 3. ENVIRONMENT - Global environment variables
    // ============================================================
    environment {
        APP_NAME = 'my-java-app'
        APP_VERSION = '1.0.0'
        MAVEN_HOME = tool name: 'maven-3.9', type: 'maven'
        JAVA_HOME = tool name: 'jdk-17', type: 'jdk'
        DOCKER_REGISTRY = 'docker.io/mycompany'
    }
    
    // ============================================================
    // 4. TOOLS - Specific tools to be available
    // ============================================================
    tools {
        maven 'maven-3.9'
        jdk 'jdk-17'
    }
    
    // ============================================================
    // 5. TRIGGERS - Automatic pipeline triggers
    // ============================================================
    triggers {
        // Build every day at 2:00 AM
        cron('0 2 * * *')
        
        // Build when changes are pushed to GitHub
        pollSCM('H/5 * * * *')
        
        // Build on specific webhook events
        // (configured via GitHub/GitLab webhooks)
    }
    
    // ============================================================
    // 6. STAGES - Main pipeline steps
    // ============================================================
    stages {
        // ------------------- STAGE 1: CHECKOUT -------------------
        stage('Checkout Source Code') {
            steps {
                echo '🔵 Cloning repository...'
                
                // Clean workspace before checkout
                cleanWs()
                
                // Checkout specific branch
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "${params.BRANCH}"]],
                    userRemoteConfigs: [[
                        url: 'https://github.com/myorg/myapp.git',
                        credentialsId: 'github-credentials'
                    ]],
                    extensions: [
                        [$class: 'CloneOption', depth: 1, noTags: true, reference: '', shallow: true],
                        [$class: 'CleanBeforeCheckout']
                    ]
                ])
                
                echo '✅ Code checked out successfully!'
            }
        }
        
        // ------------------- STAGE 2: BUILD -------------------
        stage('Build Application') {
            steps {
                echo '🟡 Building application...'
                
                // Run Maven build
                sh '''
                    mvn clean compile
                    mvn package -DskipTests=${params.SKIP_TESTS}
                '''
                
                // Archive build artifacts
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                
                echo '✅ Build completed successfully!'
            }
            
            // Post-build actions for this stage
            post {
                failure {
                    echo '❌ Build failed! Check logs.'
                }
            }
        }
        
        // ------------------- STAGE 3: TEST -------------------
        stage('Run Tests') {
            when {
                // Skip tests if parameter is true
                expression { return params.SKIP_TESTS == false }
            }
            
            steps {
                echo '🟠 Running unit and integration tests...'
                
                // Run tests with timeout
                timeout(time: 10, unit: 'MINUTES') {
                    sh 'mvn test'
                }
                
                // Publish test reports
                junit 'target/surefire-reports/*.xml'
                
                // Publish test results in UI
                publishHTML([
                    reportDir: 'target/site',
                    reportFiles: 'surefire-report.html',
                    reportName: 'Test Report'
                ])
                
                echo '✅ All tests passed!'
            }
            
            post {
                failure {
                    echo '⚠️ Some tests failed! Check reports.'
                }
            }
        }
        
        // ------------------- STAGE 4: PACKAGE -------------------
        stage('Package Application') {
            steps {
                echo '🟣 Packaging application...'
                
                // Create Docker image
                script {
                    docker.build("${DOCKER_REGISTRY}/${APP_NAME}:${APP_VERSION}")
                    
                    // Push Docker image to registry
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-credentials') {
                        docker.image("${DOCKER_REGISTRY}/${APP_NAME}:${APP_VERSION}").push()
                        docker.image("${DOCKER_REGISTRY}/${APP_NAME}:${APP_VERSION}").push('latest')
                    }
                }
                
                echo '✅ Package created and pushed!'
            }
        }
        
        // ------------------- STAGE 5: DEPLOY -------------------
        stage('Deploy to Environment') {
            steps {
                echo "🔴 Deploying to ${params.ENVIRONMENT}..."
                
                // Deploy based on environment
                script {
                    switch(params.ENVIRONMENT) {
                        case 'dev':
                            sh 'kubectl apply -f kubernetes/dev-deployment.yaml'
                            break
                        case 'staging':
                            sh 'kubectl apply -f kubernetes/staging-deployment.yaml'
                            break
                        case 'production':
                            // Manual approval required for production
                            input message: 'Deploy to production?', ok: 'Yes, deploy!'
                            sh 'kubectl apply -f kubernetes/production-deployment.yaml'
                            break
                    }
                }
                
                // Verify deployment
                sh 'kubectl rollout status deployment/myapp'
                
                echo '✅ Deployment successful!'
            }
        }
    }
    
    // ============================================================
    // 7. POST - Always run after pipeline completes
    // ============================================================
    post {
        // Always run these actions
        always {
            echo '🏁 Pipeline finished, cleaning up...'
            
            // Send notifications
            emailext(
                subject: "Pipeline ${currentBuild.currentResult}: ${env.JOB_NAME}",
                body: "Build ${env.BUILD_NUMBER} completed with status: ${currentBuild.currentResult}\\n\\nLogs: ${env.BUILD_URL}",
                to: 'team@mycompany.com',
                attachmentsPattern: 'target/*.log'
            )
            
            // Clean workspace
            cleanWs()
        }
        
        // Run on success
        success {
            echo '🎉 Pipeline completed successfully!'
            
            // Archive final artifacts
            archiveArtifacts artifacts: 'target/*.{jar,war}', fingerprint: true
            
            // Update build badge
            currentBuild.description = "✅ Version ${APP_VERSION} deployed to ${params.ENVIRONMENT}"
        }
        
        // Run on failure
        failure {
            echo '💥 Pipeline failed!'
            
            // Send Slack notification
            slackSend(
                color: 'danger',
                message: "❌ ${env.JOB_NAME} - Build #${env.BUILD_NUMBER} failed!\n${env.BUILD_URL}"
            )
            
            // Update build badge
            currentBuild.description = "❌ Failed at ${env.STAGE_NAME}"
        }
        
        // Run when build is unstable (tests failed)
        unstable {
            echo '⚠️ Build is unstable (tests failed)'
            
            slackSend(
                color: 'warning',
                message: "⚠️ ${env.JOB_NAME} - Build #${env.BUILD_NUMBER} is unstable!\n${env.BUILD_URL}"
            )
        }
        
        // Run when build is aborted
        aborted {
            echo '🛑 Build was aborted by user'
        }
    }
}


📝 Jenkinsfile - Declarative Pipeline can this be used in a Jenkinsfile ?

Yes! This Jenkinsfile CAN Be Used Directly ✅

cat > Jenkinsfile << 'EOF'
pipeline {
    // ============================================================
    // 1. AGENT - Defines where pipeline runs
    // ============================================================
    agent any  // Runs on any available Jenkins agent/worker
    
    // ============================================================
    // 2. PARAMETERS - Build-time user inputs
    // ============================================================
    parameters {
        string(name: 'BRANCH', defaultValue: 'main', description: 'Git branch to build')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'production'], description: 'Deploy target')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip unit tests')
    }
    
    // ============================================================
    // 3. ENVIRONMENT - Global environment variables
    // ============================================================
    environment {
        APP_NAME = 'my-java-app'
        APP_VERSION = '1.0.0'
        MAVEN_HOME = tool name: 'maven-3.9', type: 'maven'
        JAVA_HOME = tool name: 'jdk-17', type: 'jdk'
        DOCKER_REGISTRY = 'docker.io/mycompany'
    }
    
    // ============================================================
    // 4. TOOLS - Specific tools to be available
    // ============================================================
    tools {
        maven 'maven-3.9'
        jdk 'jdk-17'
    }
    
    // ============================================================
    // 5. TRIGGERS - Automatic pipeline triggers
    // ============================================================
    triggers {
        // Build every day at 2:00 AM
        cron('0 2 * * *')
        
        // Build when changes are pushed to GitHub
        pollSCM('H/5 * * * *')
        
        // Build on specific webhook events
        // (configured via GitHub/GitLab webhooks)
    }
    
    // ============================================================
    // 6. STAGES - Main pipeline steps
    // ============================================================
    stages {
        // ------------------- STAGE 1: CHECKOUT -------------------
        stage('Checkout Source Code') {
            steps {
                echo '🔵 Cloning repository...'
                
                // Clean workspace before checkout
                cleanWs()
                
                // Checkout specific branch
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "${params.BRANCH}"]],
                    userRemoteConfigs: [[
                        url: 'https://github.com/myorg/myapp.git',
                        credentialsId: 'github-credentials'
                    ]],
                    extensions: [
                        [$class: 'CloneOption', depth: 1, noTags: true, reference: '', shallow: true],
                        [$class: 'CleanBeforeCheckout']
                    ]
                ])
                
                echo '✅ Code checked out successfully!'
            }
        }
        
        // ------------------- STAGE 2: BUILD -------------------
        stage('Build Application') {
            steps {
                echo '🟡 Building application...'
                
                // Run Maven build
                sh '''
                    mvn clean compile
                    mvn package -DskipTests=${params.SKIP_TESTS}
                '''
                
                // Archive build artifacts
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                
                echo '✅ Build completed successfully!'
            }
            
            // Post-build actions for this stage
            post {
                failure {
                    echo '❌ Build failed! Check logs.'
                }
            }
        }
        
        // ------------------- STAGE 3: TEST -------------------
        stage('Run Tests') {
            when {
                // Skip tests if parameter is true
                expression { return params.SKIP_TESTS == false }
            }
            
            steps {
                echo '🟠 Running unit and integration tests...'
                
                // Run tests with timeout
                timeout(time: 10, unit: 'MINUTES') {
                    sh 'mvn test'
                }
                
                // Publish test reports
                junit 'target/surefire-reports/*.xml'
                
                // Publish test results in UI
                publishHTML([
                    reportDir: 'target/site',
                    reportFiles: 'surefire-report.html',
                    reportName: 'Test Report'
                ])
                
                echo '✅ All tests passed!'
            }
            
            post {
                failure {
                    echo '⚠️ Some tests failed! Check reports.'
                }
            }
        }
        
        // ------------------- STAGE 4: PACKAGE -------------------
        stage('Package Application') {
            steps {
                echo '🟣 Packaging application...'
                
                // Create Docker image
                script {
                    docker.build("${DOCKER_REGISTRY}/${APP_NAME}:${APP_VERSION}")
                    
                    // Push Docker image to registry
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-credentials') {
                        docker.image("${DOCKER_REGISTRY}/${APP_NAME}:${APP_VERSION}").push()
                        docker.image("${DOCKER_REGISTRY}/${APP_NAME}:${APP_VERSION}").push('latest')
                    }
                }
                
                echo '✅ Package created and pushed!'
            }
        }
        
        // ------------------- STAGE 5: DEPLOY -------------------
        stage('Deploy to Environment') {
            steps {
                echo "🔴 Deploying to ${params.ENVIRONMENT}..."
                
                // Deploy based on environment
                script {
                    switch(params.ENVIRONMENT) {
                        case 'dev':
                            sh 'kubectl apply -f kubernetes/dev-deployment.yaml'
                            break
                        case 'staging':
                            sh 'kubectl apply -f kubernetes/staging-deployment.yaml'
                            break
                        case 'production':
                            // Manual approval required for production
                            input message: 'Deploy to production?', ok: 'Yes, deploy!'
                            sh 'kubectl apply -f kubernetes/production-deployment.yaml'
                            break
                    }
                }
                
                // Verify deployment
                sh 'kubectl rollout status deployment/myapp'
                
                echo '✅ Deployment successful!'
            }
        }
    }
    
    // ============================================================
    // 7. POST - Always run after pipeline completes
    // ============================================================
    post {
        // Always run these actions
        always {
            echo '🏁 Pipeline finished, cleaning up...'
            
            // Send notifications
            emailext(
                subject: "Pipeline ${currentBuild.currentResult}: ${env.JOB_NAME}",
                body: "Build ${env.BUILD_NUMBER} completed with status: ${currentBuild.currentResult}\\n\\nLogs: ${env.BUILD_URL}",
                to: 'team@mycompany.com',
                attachmentsPattern: 'target/*.log'
            )
            
            // Clean workspace
            cleanWs()
        }
        
        // Run on success
        success {
            echo '🎉 Pipeline completed successfully!'
            
            // Archive final artifacts
            archiveArtifacts artifacts: 'target/*.{jar,war}', fingerprint: true
            
            // Update build badge
            currentBuild.description = "✅ Version ${APP_VERSION} deployed to ${params.ENVIRONMENT}"
        }
        
        // Run on failure
        failure {
            echo '💥 Pipeline failed!'
            
            // Send Slack notification
            slackSend(
                color: 'danger',
                message: "❌ ${env.JOB_NAME} - Build #${env.BUILD_NUMBER} failed!\n${env.BUILD_URL}"
            )
            
            // Update build badge
            currentBuild.description = "❌ Failed at ${env.STAGE_NAME}"
        }
        
        // Run when build is unstable (tests failed)
        unstable {
            echo '⚠️ Build is unstable (tests failed)'
            
            slackSend(
                color: 'warning',
                message: "⚠️ ${env.JOB_NAME} - Build #${env.BUILD_NUMBER} is unstable!\n${env.BUILD_URL}"
            )
        }
        
        // Run when build is aborted
        aborted {
            echo '🛑 Build was aborted by user'
        }
    }
}EOF



📋 Method 3: Using Echo Command

cat > Jenkinsfile << 'EOF'
pipeline {
    // ============================================================
    // 1. AGENT - Defines where pipeline runs
    // ============================================================
    agent any  // Runs on any available Jenkins agent/worker
    
    // ============================================================
    // 2. PARAMETERS - Build-time user inputs
    // ============================================================
    parameters {
        string(name: 'BRANCH', defaultValue: 'main', description: 'Git branch to build')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'production'], description: 'Deploy target')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip unit tests')
    }
    
    // ============================================================
    // 3. ENVIRONMENT - Global environment variables
    // ============================================================
    environment {
        APP_NAME = 'my-java-app'
        APP_VERSION = '1.0.0'
        MAVEN_HOME = tool name: 'maven-3.9', type: 'maven'
        JAVA_HOME = tool name: 'jdk-17', type: 'jdk'
        DOCKER_REGISTRY = 'docker.io/mycompany'
    }
    
    // ============================================================
    // 4. TOOLS - Specific tools to be available
    // ============================================================
    tools {
        maven 'maven-3.9'
        jdk 'jdk-17'
    }
    
    // ============================================================
    // 5. TRIGGERS - Automatic pipeline triggers
    // ============================================================
    triggers {
        // Build every day at 2:00 AM
        cron('0 2 * * *')
        
        // Build when changes are pushed to GitHub
        pollSCM('H/5 * * * *')
        
        // Build on specific webhook events
        // (configured via GitHub/GitLab webhooks)
    }
    
    // ============================================================
    // 6. STAGES - Main pipeline steps
    // ============================================================
    stages {
        // ------------------- STAGE 1: CHECKOUT -------------------
        stage('Checkout Source Code') {
            steps {
                echo '🔵 Cloning repository...'
                
                // Clean workspace before checkout
                cleanWs()
                
                // Checkout specific branch
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "${params.BRANCH}"]],
                    userRemoteConfigs: [[
                        url: 'https://github.com/myorg/myapp.git',
                        credentialsId: 'github-credentials'
                    ]],
                    extensions: [
                        [$class: 'CloneOption', depth: 1, noTags: true, reference: '', shallow: true],
                        [$class: 'CleanBeforeCheckout']
                    ]
                ])
                
                echo '✅ Code checked out successfully!'
            }
        }
        
        // ------------------- STAGE 2: BUILD -------------------
        stage('Build Application') {
            steps {
                echo '🟡 Building application...'
                
                // Run Maven build
                sh '''
                    mvn clean compile
                    mvn package -DskipTests=${params.SKIP_TESTS}
                '''
                
                // Archive build artifacts
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                
                echo '✅ Build completed successfully!'
            }
            
            // Post-build actions for this stage
            post {
                failure {
                    echo '❌ Build failed! Check logs.'
                }
            }
        }
        
        // ------------------- STAGE 3: TEST -------------------
        stage('Run Tests') {
            when {
                // Skip tests if parameter is true
                expression { return params.SKIP_TESTS == false }
            }
            
            steps {
                echo '🟠 Running unit and integration tests...'
                
                // Run tests with timeout
                timeout(time: 10, unit: 'MINUTES') {
                    sh 'mvn test'
                }
                
                // Publish test reports
                junit 'target/surefire-reports/*.xml'
                
                // Publish test results in UI
                publishHTML([
                    reportDir: 'target/site',
                    reportFiles: 'surefire-report.html',
                    reportName: 'Test Report'
                ])
                
                echo '✅ All tests passed!'
            }
            
            post {
                failure {
                    echo '⚠️ Some tests failed! Check reports.'
                }
            }
        }
        
        // ------------------- STAGE 4: PACKAGE -------------------
        stage('Package Application') {
            steps {
                echo '🟣 Packaging application...'
                
                // Create Docker image
                script {
                    docker.build("${DOCKER_REGISTRY}/${APP_NAME}:${APP_VERSION}")
                    
                    // Push Docker image to registry
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-credentials') {
                        docker.image("${DOCKER_REGISTRY}/${APP_NAME}:${APP_VERSION}").push()
                        docker.image("${DOCKER_REGISTRY}/${APP_NAME}:${APP_VERSION}").push('latest')
                    }
                }
                
                echo '✅ Package created and pushed!'
            }
        }
        
        // ------------------- STAGE 5: DEPLOY -------------------
        stage('Deploy to Environment') {
            steps {
                echo "🔴 Deploying to ${params.ENVIRONMENT}..."
                
                // Deploy based on environment
                script {
                    switch(params.ENVIRONMENT) {
                        case 'dev':
                            sh 'kubectl apply -f kubernetes/dev-deployment.yaml'
                            break
                        case 'staging':
                            sh 'kubectl apply -f kubernetes/staging-deployment.yaml'
                            break
                        case 'production':
                            // Manual approval required for production
                            input message: 'Deploy to production?', ok: 'Yes, deploy!'
                            sh 'kubectl apply -f kubernetes/production-deployment.yaml'
                            break
                    }
                }
                
                // Verify deployment
                sh 'kubectl rollout status deployment/myapp'
                
                echo '✅ Deployment successful!'
            }
        }
    }
    
    // ============================================================
    // 7. POST - Always run after pipeline completes
    // ============================================================
    post {
        // Always run these actions
        always {
            echo '🏁 Pipeline finished, cleaning up...'
            
            // Send notifications
            emailext(
                subject: "Pipeline ${currentBuild.currentResult}: ${env.JOB_NAME}",
                body: "Build ${env.BUILD_NUMBER} completed with status: ${currentBuild.currentResult}\\n\\nLogs: ${env.BUILD_URL}",
                to: 'team@mycompany.com',
                attachmentsPattern: 'target/*.log'
            )
            
            // Clean workspace
            cleanWs()
        }
        
        // Run on success
        success {
            echo '🎉 Pipeline completed successfully!'
            
            // Archive final artifacts
            archiveArtifacts artifacts: 'target/*.{jar,war}', fingerprint: true
            
            // Update build badge
            currentBuild.description = "✅ Version ${APP_VERSION} deployed to ${params.ENVIRONMENT}"
        }
        
        // Run on failure
        failure {
            echo '💥 Pipeline failed!'
            
            // Send Slack notification
            slackSend(
                color: 'danger',
                message: "❌ ${env.JOB_NAME} - Build #${env.BUILD_NUMBER} failed!\n${env.BUILD_URL}"
            )
            
            // Update build badge
            currentBuild.description = "❌ Failed at ${env.STAGE_NAME}"
        }
        
        // Run when build is unstable (tests failed)
        unstable {
            echo '⚠️ Build is unstable (tests failed)'
            
            slackSend(
                color: 'warning',
                message: "⚠️ ${env.JOB_NAME} - Build #${env.BUILD_NUMBER} is unstable!\n${env.BUILD_URL}"
            )
        }
        
        // Run when build is aborted
        aborted {
            echo '🛑 Build was aborted by user'
        }
    }
}
EOF
