pipeline {
    agent any

    environment {
        SONAR_TOKEN = credentials('calculator-token')
        DATADOG_API_KEY = credentials('datadog')
    }

    tools {
        jdk 'jdk-17'
        nodejs 'nodejs-14'
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            steps {
                script {
                    bat 'docker rmi -f myapp:latest || true'
                    bat 'npm install'
                    bat 'docker build -t myapp:latest .'
                }
            }
        }
        stage('Test') {
            steps {
                script {
                    bat 'set NODE_ENV=test && set PORT=4000 && npm test'
                }
            }
        }
        
        stage('Deploy') {
            steps {
                script {
                    bat 'docker stop myapp-container || true'
                    bat 'docker rm myapp-container || true'
                    bat 'docker run -d --name myapp-container -p 3000:3000 myapp:latest'
                }
            }
        }
        stage('Monitoring and Alerting') {
            steps {
                script {
                    def startTime = currentBuild.startTimeInMillis
                    def currentTime = System.currentTimeMillis()
                    def duration = (currentTime - startTime) / 1000
                    def user = currentBuild.rawBuild.getCauses().find { it.userName }?.userName ?: "Automated Trigger"

                    withCredentials([string(credentialsId: 'datadog', variable: 'DATADOG_API_KEY')]) {
                        def response = httpRequest (
                            url: "https://api.us5.datadoghq.com/api/v1/events",
                            httpMode: 'POST',
                            customHeaders: [[name: 'Content-Type', value: 'application/json'], [name: 'DD-API-KEY', value: "${DATADOG_API_KEY}"]],
                            requestBody: """{
                                "title": "Deployment Notification",
                                "text": "Deployment of myapp to production was successful.\\nBuild Duration: ${duration} seconds\\nTriggered by: ${user}",
                                "priority": "normal",
                                "tags": ["jenkins","deployment","myapp"]
                            }"""
                        )
                        echo "Datadog event response: ${response.content}"
                    }
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
