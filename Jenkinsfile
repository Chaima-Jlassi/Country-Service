pipeline {
    agent any

    tools { maven 'mymaven' }

    environment {
        JAVA_HOME = '/opt/jdk-21'
        PATH = "/opt/jdk-21/bin:${env.PATH}"
    }

    stages {
        stage('Checkout code') {
            steps { checkout scm }
        }

        stage('Compile, test, package') {
            steps { sh 'mvn clean install' }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

     stage('Deploy to Kubernetes') {
    steps {
        script {
            kubeconfig(credentialsId: 'kubeconfig-file', serverUrl: '') {
                sh 'kubectl apply -f deployment.yaml'
                sh 'kubectl apply -f service.yaml'
            }
        }
    }
        }

      

       
            }
        }
    }
}
