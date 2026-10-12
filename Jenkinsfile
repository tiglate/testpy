pipeline {
    agent any
    triggers {
        // Checks for new commits/merges every 1 minute
        pollSCM('* * * * *')
    }
    environment {
        REGISTRY = 'localhost:5000'
        IMAGE_NAME = 'python-web-app'
        IMAGE_TAG = "${env.BUILD_ID}" // Unique tag for every build
    }
    stages {
        stage('Unit Tests & Coverage') {
            steps {
                // Uses an ephemeral container to test, keeping Jenkins clean
                sh '''
                docker run --rm -v $(pwd):/app -w /app python:3.9-slim bash -c "
                    pip install -r requirements.txt
                    pytest --cov=./ --cov-report=term --cov-fail-under=80
                "
                '''
            }
        }
        stage('Qodana Code Quality') {
            steps {
                // Runs JetBrains Qodana. Fails the pipeline if any critical issues are found.
                sh '''
                docker run --rm -v $(pwd):/data/project/ \
                  jetbrains/qodana-python-community:2026.2 \
                  --fail-threshold 0
                '''
            }
        }
        stage('Build Docker Image') {
            steps {
                // Builds and tags the image with both the specific build ID and 'latest'
                sh "docker build -t ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} -t ${REGISTRY}/${IMAGE_NAME}:latest ."
            }
        }
        stage('Push to Registry') {
            steps {
                // Archives the images to your local registry
                sh "docker push ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
                sh "docker push ${REGISTRY}/${IMAGE_NAME}:latest"
            }
        }
        stage('Deploy Application') {
            steps {
                // Removes the old container and starts the newly built one
                sh '''
                docker stop prod-python-app || true
                docker rm prod-python-app || true
                docker run -d -p 8081:5000 --name prod-python-app ${REGISTRY}/${IMAGE_NAME}:latest
                '''
            }
        }
    }
}
