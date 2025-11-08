pipeline {
    agent any

    environment {
        SONAR_HOST_URL = 'http://192.168.50.4:9000/'
        SONAR_AUTH_TOKEN = credentials('sonarqube')  // Make sure this is the correct credentials ID
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Static Code Analysis (Bandit + SonarQube)') {
            steps {
                // Bandit security scan
                sh '''
                    echo "Running Bandit static analysis..."
                    pip install --user bandit
                    bandit -r . -f html -o bandit-report.html || true
                '''

                // SonarQube scan via Jenkins plugin
                withSonarQubeEnv('SonarQube') { // The name must match your Jenkins SonarQube configuration
                    sh '''
                        echo "Running SonarQube analysis using Jenkins plugin..."
                        sonar-scanner \
                            -Dsonar.projectKey=PythonApp \
                            -Dsonar.projectName=PythonApp \
                            -Dsonar.projectVersion=1.0 \
                            -Dsonar.sources=. \
                            -Dsonar.python.version=3.11
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    echo "Building Docker image..."
                    docker build -t python-app:latest .
                '''
            }
        }

        stage('Dynamic Application Security Testing (OWASP ZAP)') {
            steps {
                sh '''
                    echo "Starting OWASP ZAP scan..."
                    docker run --rm -v $(pwd):/zap/wrk/:rw -t owasp/zap2docker-stable zap-baseline.py \
                        -t http://localhost:5000 -r zap-report.html || true
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    echo "Deploying application..."
                    docker run -d -p 5000:5000 python-app:latest
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '*.html', allowEmptyArchive: true
            emailext(
                subject: "DevSecOps Pipeline - ${currentBuild.currentResult}",
                body: "The pipeline finished with status: ${currentBuild.currentResult}\nCheck attached reports.",
                to: 'yahyakaddour5@gmail.com',
                attachmentsPattern: '*.html'
            )
        }
    }
}
