pipeline {
    agent any

    tools { maven 'mymaven' }

    environment {
        JAVA_HOME = '/opt/jdk-21'
        PATH = "/opt/jdk-21/bin:${env.PATH}"
        DOCKER_USER = 'chaymajlassi'
        IMAGE = "chaymajlassi/my-country-service"
    }

    stages {
        stage('Checkout code') {
            steps { checkout scm }
        }

        stage('Build maven') {
            steps { sh 'mvn clean install' }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Build Dockerfile') {
            steps {
                sh 'docker build . -t ${IMAGE}:${BUILD_NUMBER}'
                withCredentials([string(credentialsId: 'dockerhub-pwd', variable: 'dockerhubpwd')]) {
                    sh 'echo "$dockerhubpwd" | docker login -u ${DOCKER_USER} --password-stdin'
                }
                sh 'docker push ${IMAGE}:${BUILD_NUMBER}'
            }
        }

        stage('Deploy micro-service') {
            steps {
                // remove only this app's container, not every container on the host
                sh 'docker rm -f country-service || true'
                // host port 8083 because Nexus already uses 8082 on this machine
                sh 'docker run -d --name country-service -p 8083:8082 ${IMAGE}:${BUILD_NUMBER}'
            }
        }
    }
}
