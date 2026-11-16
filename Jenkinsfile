pipeline {
    agent any

    tools {
        maven 'mymaven'
    }

    environment {
        JAVA_HOME     = '/opt/jdk-21'
        PATH          = "/opt/jdk-21/bin:${env.PATH}"
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

        stage('Deploy with Ansible') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig-file',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        ansible-playbook -i localhost, playbookCICD.yml \
                            -e image_name=${IMAGE_NAME} \
                            -e image_tag=${BUILD_NUMBER} \
                            -e kubeconfig_path=${KUBECONFIG_FILE}
                    '''
                }
            }
        }

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
            echo 'Pipeline OK: application deployed with Ansible + kube-state-metrics running.'
        }
        failure {
            echo 'Pipeline failed - check the stage logs above.'
        }
    }
}
