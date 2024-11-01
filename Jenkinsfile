pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = 'yourdockerhub/spotify-clone'
        DOCKER_TAG = "${BUILD_NUMBER}"
        SONAR_PROJECT_KEY = 'spotify-clone'
        REGISTRY_CREDENTIALS = credentials('dockerhub-cred')
        SONARQUBE_TOKEN = credentials('sonarqube-token')
    }
    
    tools {
        nodejs 'NodeJS 20.x'
    }

    stages {
        stage('Git Checkout') {
            steps {
                cleanWs()
                git branch: 'main', url: 'https://github.com/yourusername/spotify-clone.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        sonar-scanner \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.sources=src \
                        -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                        -Dsonar.host.url=http://your-sonarqube-url:9000 \
                        -Dsonar.login=${SONARQUBE_TOKEN}
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '''
                    --scan ./ 
                    --format HTML 
                    --format XML 
                    --prettyPrint
                    ''',
                    odcInstallation: 'OWASP-DC'
                
                dependencyCheckPublisher pattern: 'dependency-check-report.xml'
            }
        }

        stage('Trivy File Scan') {
            steps {
                sh '''
                    trivy fs . \
                    --security-checks vuln,config,secret \
                    --severity HIGH,CRITICAL \
                    --exit-code 1
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                }
            }
        }

        stage('Tag & Push to DockerHub') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-cred') {
                        def image = docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}")
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }

        stage('Docker Scout Scan') {
            steps {
                sh """
                    docker scout cves ${DOCKER_IMAGE}:${DOCKER_TAG} \
                    --exit-code \
                    --only-severity critical,high
                """
            }
        }

        stage('Deploy to Container') {
            steps {
                script {
                    sh """
                        docker stop spotify-clone || true
                        docker rm spotify-clone || true
                        docker run -d \
                            --name spotify-clone \
                            --restart unless-stopped \
                            -p 3000:3000 \
                            ${DOCKER_IMAGE}:${DOCKER_TAG}
                    """
                }
            }
        }
    }

    post {
        always {
            cleanWs()
            sh 'docker system prune -f'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}