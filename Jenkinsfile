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
                sh '''
                    echo "Updating system..."
                    apt-get update -y && apt-get install -y curl

                    echo "Installing Bandit..."
                    pip install bandit

                    echo "Running Bandit static analysis..."
                    mkdir -p reports
                    bandit -r . -f html -o reports/bandit-report.html || true

                    echo "Running SonarQube analysis via Jenkins plugin..."
                '''
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        sonar-scanner \
                            -Dsonar.projectKey=PythonApp \
                            -Dsonar.projectName=PythonApp \
                            -Dsonar.projectVersion=1.0 \
                            -Dsonar.sources=. \
                            -Dsonar.python.version=3.11 \
                            -Dsonar.host.url=$SONAR_HOST_URL \
                            -Dsonar.login=$SONAR_AUTH_TOKEN
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building Docker image..."
                    docker build -t bankapp .
                '''
            }
        }

        stage('Deploy and Run App') {
            steps {
                sh '''
                    echo "Running application container..."
                    docker run -d --name bankapp -p 5000:5000 bankapp
                '''
            }
        }

        stage('Dynamic Analysis (OWASP ZAP)') {
            steps {
                sh '''
                    echo "Starting OWASP ZAP scan..."
                    zap-cli quick-scan --self-contained --start-options "-config api.disablekey=true" http://localhost:5000 || true
                    mkdir -p reports
                    zap-cli report -o reports/zap-report.html -f html || true
                '''
            }
        }
    }

    post {
        always {
            sh '''
                echo "Cleaning up..."
                docker stop bankapp || true
                docker rm bankapp || true
            '''

            archiveArtifacts artifacts: 'reports/*.html', allowEmptyArchive: true

            emailext(
                subject: "Jenkins Pipeline Report: ${currentBuild.currentResult}",
                body: "Hello,\n\nThe Jenkins DevSecOps pipeline has completed with status: ${currentBuild.currentResult}.\nReports are attached.\n\nBest,\nJenkins",
                attachLog: true,
                to: 'yahyakaddour5@gmail.com'
            )
        }
    }
}
