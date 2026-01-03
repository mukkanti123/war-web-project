pipeline {
    agent any

    environment {
        // -------- SonarQube --------
        SONAR_HOST_URL = "http://65.0.128.46:9000"

        // -------- Nexus --------
        NEXUS_URL = "65.2.6.21:8081"
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_CREDENTIAL_ID = "nexus_creds"

        // -------- Tomcat --------
        TOMCAT_URL = "http://13.203.74.89:8080"
        TOMCAT_CONTEXT = "wwp"
    }

    tools {
        maven "maven"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
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
                withCredentials([
                    string(credentialsId: 'sonar_token', variable: 'SONAR_TOKEN')
                ]) {
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

        stage('Extract Version') {
            steps {
                script {
                    env.ART_VERSION = sh(
                        script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                        returnStdout: true
                    ).trim()
                    echo "📦 Artifact Version: ${ART_VERSION}"
                }
            }
        }

        stage('Publish to Nexus') {
            steps {
                script {
                    def warFile = sh(
                        script: "find target -name '*.war' -print -quit",
                        returnStdout: true
                    ).trim()

                    def releaseVersion = "${ART_VERSION}-${BUILD_NUMBER}"

                    echo "🚀 Uploading WAR to Nexus"

                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: NEXUS_URL,
                        repository: NEXUS_REPOSITORY,
                        credentialsId: NEXUS_CREDENTIAL_ID,
                        groupId: 'koddas.web.war',
                        version: releaseVersion,
                        artifacts: [[
                            artifactId: 'wwp',
                            classifier: '',
                            file: warFile,
                            type: 'war'
                        ]]
                    )
                }
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                echo "🚀 Deploying WAR to Tomcat..."
                sshagent(credentials: ['tomcat_ssh_key']) {
                    sh """
                        WAR_FILE=\$(find target -name '*.war' -print -quit)

                        scp -o StrictHostKeyChecking=no \$WAR_FILE ubuntu@13.203.74.89:/tmp/wwp.war

                        ssh -o StrictHostKeyChecking=no ubuntu@13.203.74.89 '
                            sudo rm -rf /opt/tomcat/webapps/wwp*
                            sudo mv /tmp/wwp.war /opt/tomcat/webapps/
                            sudo systemctl restart tomcat
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully"
            echo "🔗 SonarQube: ${SONAR_HOST_URL}/dashboard?id=wwp"
            echo "📦 Nexus: http://${NEXUS_URL}"
            echo "🌐 App URL: ${TOMCAT_URL}/${TOMCAT_CONTEXT}"
        }

        failure {
            echo "❌ Pipeline failed — check logs"
        }

        always {
            echo "🧹 Pipeline execution finished"
        }
    }
}
