pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = 'dockerhub_creds'
        DOCKER_IMAGE = '9346278398/wwp:1.0'
        SONAR_HOST_URL = "http://65.0.128.46:9000"
    }

    tools { maven "maven" }

    stages {

        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build WAR') {
            steps {
                echo "🔨 Building WAR..."
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: 'target/*.war'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "🔍 Running SonarQube analysis..."
                withCredentials([string(credentialsId: 'sonar_token', variable: 'SONAR_TOKEN')]) {
                    sh """
                        mvn sonar:sonar \
                        -Dsonar.projectKey=wwp \
                        -Dsonar.host.url=${SONAR_HOST_URL} \
                        -Dsonar.token=${SONAR_TOKEN} \
                        -Dsonar.java.binaries=target/classes
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🐳 Building Docker image..."
                sh "docker build -t ${DOCKER_IMAGE} ."
            }
        }

        stage('Push Docker Image') {
            steps {
                echo "📤 Pushing Docker image to Docker Hub..."
                withCredentials([usernamePassword(credentialsId: DOCKERHUB_CREDENTIALS, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    sh "docker push ${DOCKER_IMAGE}"
                }
            }
        }

        stage('Run Docker Container') {
            steps {
                echo "🚀 Running Docker container..."
                sh "docker rm -f wwp-app || true"
                sh "docker run -d --name wwp-app -p 8080:8080 ${DOCKER_IMAGE}"
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully"
            echo "🔗 SonarQube: ${SONAR_HOST_URL}/dashboard?id=wwp"
            echo "🌐 Docker Container: wwp-app running on port 8080"
        }

        failure { echo "❌ Pipeline failed — check logs" }

        always { echo "🧹 Pipeline execution finished" }
    }
}
