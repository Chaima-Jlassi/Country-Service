pipeline {
    agent any

    tools {
        maven 'mymaven'
    }

    environment {
        JAVA_HOME = '/opt/jdk-21'
        PATH = "/opt/jdk-21/bin:${env.PATH}"
        MONITORING_NS = 'monitoring'
        KSM_RELEASE   = 'kube-state-metrics'
        IMAGE_NAME    = 'my-country-service'
    }

    stages {
        stage('Checkout code') {
            steps {
                checkout scm
            }
        }

        stage('Compile, test, package') {
            steps {
                sh 'mvn clean install'
            }
            post {
                always {
                    junit allowEmptyResults: true,
                          testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        // Chapter 8: Ansible builds the Docker image, pushes it to Docker Hub
        // and applies deployment.yaml / service.yaml (see playbookCICD.yml)
       stage('Build, push and deploy with Ansible') {
    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'dockerhub-creds',
                usernameVariable: 'DOCKER_USER',
                passwordVariable: 'DOCKER_PASS'
            ),
            string(
                credentialsId: 'kubernetes-pwd',   // Secret text
                variable: 'KUBECONFIG_CONTENT'
            )
        ]) {
            sh '''
                # Write the secret text into a real file
                echo "$KUBECONFIG_CONTENT" > kubeconfig.tmp
                chmod 600 kubeconfig.tmp

                ansible-playbook -i localhost, playbookCICD.yml \
                    -e image_name=${IMAGE_NAME} \
                    -e image_tag=${BUILD_NUMBER} \
                    -e docker_registry_username=${DOCKER_USER} \
                    -e docker_registry_password=${DOCKER_PASS} \
                    -e kubeconfig_path=$(pwd)/kubeconfig.tmp
            '''
        }
    }
}

        // Kept from the original pipeline (idempotent if Ansible already applied them)
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

        // Chapter 9: monitoring (Prometheus scrapes it, Grafana dashboard 13332)
        stage('Deploy kube-state-metrics (Helm)') {
            steps {
                script {
                    kubeconfig(credentialsId: 'kubeconfig-file', serverUrl: '') {
                        sh '''
                            helm repo add prometheus-community https://prometheus-community.github.io/helm-charts || true
                            helm repo update
                            helm upgrade --install ${KSM_RELEASE} prometheus-community/kube-state-metrics \
                                --namespace ${MONITORING_NS} --create-namespace \
                                --set service.type=NodePort \
                                --set service.nodePort=30080 \
                                --wait --timeout 120s
                        '''
                    }
                }
            }
        }

        stage('Verify monitoring') {
            steps {
                script {
                    kubeconfig(credentialsId: 'kubeconfig-file', serverUrl: '') {
                        sh '''
                            kubectl rollout status deployment/${KSM_RELEASE} -n ${MONITORING_NS} --timeout=120s
                            kubectl get pods -n ${MONITORING_NS}
                            kubectl get pods
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline OK: app deployed with Ansible, kube-state-metrics running. Check Grafana dashboard 13332.'
        }
        failure {
            echo 'Pipeline failed - check the stage logs above.'
        }
    }
}
