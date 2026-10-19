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

        stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv(installationName: 'MySonarQubeServer', credentialsId: 'sonarqubePWD') {
            sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=country-service -Dsonar.projectName=country-service'
        }
    }
}
        stage('Publish artifact to Nexus') {
    steps { sh 'mvn deploy -DskipTests' }
}

        stage('Store reports in Nexus') {
    steps {
       withCredentials([usernamePassword(credentialsId: 'nexus-creds',
                                  usernameVariable: 'NEXUS_USER',
                                  passwordVariable: 'NEXUS_PASS')]) {
            sh '''
                tar -czf jacoco-report.tar.gz target/site/jacoco
                tar -czf surefire-reports.tar.gz target/surefire-reports
                for f in jacoco-report.tar.gz surefire-reports.tar.gz; do
                  curl -sf -u "$NEXUS_USER:$NEXUS_PASS" --upload-file "$f" \
                    "http://localhost:8082/repository/reports/country-service/${BUILD_NUMBER}/$f"
                done
            '''
        }
    }
}

        
}
