pipeline {
    agent {
        docker {
            image 'python:3.11-slim'
            args '-u root'
        }
    }

    environment {
        SONAR_HOST_URL = 'http://192.168.50.4:9000/'
        SONAR_AUTH_TOKEN = credentials('sonarqube')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Static Code Analysis (Bandit + SonarQube)') {
            steps {
                script {
                    try {
                        sh '''
                            echo "🔧 Updating system..."
                            apt-get update -y && apt-get install -y curl

                            echo "📦 Installing Bandit..."
                            pip install bandit

                            echo "🔍 Running Bandit static analysis..."
                            mkdir -p reports
                            bandit -r . -f html -o reports/bandit-report.html || true
                        '''
                        echo "🚀 Running SonarQube analysis..."
                        withSonarQubeEnv('SonarQube') {
                            sh '''
                                sonar-scanner \
                                    -Dsonar.projectKey=PythonApp \
                                    -Dsonar.projectName=PythonApp \
                                    -Dsonar.projectVersion=1.0 \
                                    -Dsonar.sources=. \
                                    -Dsonar.python.version=3.11 \
                                    -Dsonar.host.url=$SONAR_HOST_URL \
                                    -Dsonar.login=$SONAR_AUTH_TOKEN || true
                            '''
                        }
                    } catch (err) {
                        echo "⚠️ Static analysis failed: ${err}"
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    try {
                        sh '''
                            echo "🐳 Building Docker image..."
                            docker build -t bankapp .
                        '''
                    } catch (err) {
                        echo "⚠️ Docker build failed: ${err}"
                    }
                }
            }
        }

        stage('Deploy and Run App') {
            steps {
                script {
                    try {
                        sh '''
                            echo "🚀 Running application container..."
                            docker run -d --name bankapp -p 5000:5000 bankapp
                        '''
                    } catch (err) {
                        echo "⚠️ Failed to run app container: ${err}"
                    }
                }
            }
        }

        stage('Dynamic Analysis (OWASP ZAP)') {
            steps {
                script {
                    try {
                        sh '''
                            echo "🧪 Starting OWASP ZAP scan..."
                            zap-cli quick-scan --self-contained --start-options "-config api.disablekey=true" http://localhost:5000 || true
                            mkdir -p reports
                            zap-cli report -o reports/zap-report.html -f html || true
                        '''
                    } catch (err) {
                        echo "⚠️ OWASP ZAP scan failed: ${err}"
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                echo "🧹 Cleaning up containers..."
                sh '''
                    docker stop bankapp || true
                    docker rm bankapp || true
                '''
            }

            archiveArtifacts artifacts: 'reports/*.html', allowEmptyArchive: true

            emailext(
                subject: "🛡️ DevSecOps Pipeline Report: ${currentBuild.currentResult}",
                body: """Hello,<br><br>
                The Jenkins DevSecOps pipeline has completed with status: <b>${currentBuild.currentResult}</b>.<br>
                Reports (Bandit + OWASP ZAP) are archived in Jenkins.<br><br>
                Best,<br>Jenkins""",
                mimeType: 'text/html',
                attachLog: true,
                to: 'yahyakaddour5@gmail.com'
            )
        }
    }
}
