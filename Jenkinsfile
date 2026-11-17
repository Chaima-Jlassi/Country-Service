
pipeline {
    agent any

    tools {
        maven 'mymaven'
    }

    environment {
    JAVA_HOME     = '/opt/jdk-21'
    PATH          = "/usr/local/bin:/opt/jdk-21/bin:${env.PATH}"
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
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv(installationName: 'MySonarQubeServer', credentialsId: 'sonarqubePWD') {
                    sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=country-service -Dsonar.projectName=country-service'
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
                            -e "image_name=${IMAGE_NAME}" \
                            -e "image_tag=${BUILD_NUMBER}" \
                            -e "kubeconfig_path=${KUBECONFIG_FILE}"
                    '''
                }
            }
        }

        stage('Deploy kube-state-metrics (Helm)') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig-file',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        set -eux

                        export KUBECONFIG="$KUBECONFIG_FILE"

                        echo "Checking Helm..."
                        helm version

                        echo "Checking Kubernetes..."
                        kubectl get nodes

                        echo "Adding Prometheus repository..."
                        helm repo add prometheus-community \
                            https://prometheus-community.github.io/helm-charts \
                            --force-update

                        helm repo update

                        echo "Creating monitoring namespace..."
                        kubectl create namespace ${MONITORING_NS} \
                            --dry-run=client -o yaml | kubectl apply -f -

                        echo "Deploying kube-state-metrics..."
                        helm upgrade --install ${KSM_RELEASE} \
                            prometheus-community/kube-state-metrics \
                            --namespace ${MONITORING_NS} \
                            --set service.type=NodePort \
                            --set service.nodePort=30080 \
                            --wait \
                            --timeout 180s

                        echo "kube-state-metrics deployed successfully!"
                    '''
                }
            }
        }

        stage('Verify monitoring') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig-file',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        set -eux

                        export KUBECONFIG="$KUBECONFIG_FILE"

                        echo "Checking deployments..."
                        kubectl get deployments -n ${MONITORING_NS}

                        echo "Checking pods..."
                        kubectl get pods -n ${MONITORING_NS}

                        echo "Checking services..."
                        kubectl get services -n ${MONITORING_NS}

                        echo "Checking rollout..."
                        kubectl rollout status \
                            deployment/${KSM_RELEASE} \
                            -n ${MONITORING_NS} \
                            --timeout=180s

                        echo "Monitoring verification successful!"
                    '''
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
