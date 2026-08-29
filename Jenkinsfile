pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh '''
                    echo "Starting build process..."
                    mkdir -p build/output
                    echo "Build artifacts created in build/output/" > build/output/build_info.txt
                    ls -la build/output/
                '''
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: 'build/output/*'
            echo 'Build completed successfully!'
        }
        failure {
            echo 'Build failed!'
        }
    }
}
