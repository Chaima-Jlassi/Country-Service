pipeline {
    agent any
    tools {
        maven 'mymaven'
    }
    stages {
        stage('Checkout code')
        {
            steps {
                git branch: 'Master', url: 'https://github.com/Chaima-Jlassi/Country-Service.git'
            }
        }
        stage('Build maven')
        {
            steps {
                sh 'mvn clean install'
            }
        }
        stage('Build Dockerfile ')
        {
            steps {
                sh 'docker build . -t my-country-service:$BUILD_NUMBER '
                withCredentials([string(credentialsId: 'dockerhub-pwd', variable: 'dockerhubpwd')]) {
                sh 'docker login -u chaymajlassi -p ${dockerhubpwd}'
                }
                sh 'docker tag my-country-service:$BUILD_NUMBER jamina2385/my-country-service:$BUILD_NUMBER'
                sh 'docker push jamina2385/my-country-service:$BUILD_NUMBER'
            }
        }
        stage('Deploy micro-service')
        {
            steps {
                sh 'docker rm -f $(docker ps -aq)'
                sh 'docker run -d -p 8082:8082 my-country-service:$BUILD_NUMBER'
            }
        }
    }
}
