pipeline {
    agent any

    environment {
       SONAR_TOKEN = credentials('sonar-analysis')
       SONAR_PROJECT_KEY = 'owasp-juice-shop'
       DOCKER_IMAGE_NAME = 'owasp-juice-shop'
       NEXUS_DOCKER_REGISTRY = '<your_dsb_hub_ip_address>:8082'
       NEXUS_DOCKER_PUSH_INDEX = '<your_dsb_hub_ip_address>:8083'
       NEXUS_DOCKER_PUSH_PATH = 'repository/docker-host'
       target_name= 'https://owasp.org/www-project-juice-shop/' // This is used in ZAP Scan for the target URL, change it to your deployed instance if needed
    }

    stages {
        stage('Clone') {
            steps {
                checkout scmGit(branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[credentialsId: 'Gitea PAT', url: 'http://<your_dsb_hub_ip_address>/<your_username>/owasp-juice-shop.git']])
            }
        }
        stage('Build') {
            steps {
                sh 'docker build -t ${DOCKER_IMAGE_NAME}:${BUILD_NUMBER} .'
            }
        }
        stage('Security Scan'){
            parallel {
                stage('Sonar Scan') {
                    steps {
                        script {
                            try{
                                withSonarQubeEnv(installationName: 'Sonar Server', credentialsId: 'sonar-analysis') {
                                    sh '''
                                    docker run --rm \
                                    -e SONAR_HOST_URL="${SONAR_HOST_URL}" \
                                    -e SONAR_TOKEN="${SONAR_TOKEN}" \
                                    -v "$(pwd):/usr/src" \
                                    sonarsource/sonar-scanner-cli \
                                    -Dsonar.projectKey="${SONAR_PROJECT_KEY}" \
                                    -Dsonar.qualitygate.wait=true \
                                    -Dsonar.sources=.
                                    '''
                                }
                            } catch (Exception e) {
                                // Handle the error
                                echo "Quality Qate check has failed: ${e}"
                                currentBuild.result = 'UNSTABLE' // Mark the build as unstable instead of failing
                            }
                        }
                    }
                }
                stage('Security Scan') {
                    steps {
                        sh '''
                            trivy image --severity HIGH,CRITICAL ${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}
                        '''
                    }
                }
            }
                    stage('Security Scan'){
                        steps {
                            script {
                                echo "Starting Security Scans"

                                sh '''
                                    docker run --rm -t ghcr.io/zaproxy/zaproxy:stable zap-full-scan.py \
                                    -t ${target_name} \
                                    -r zap_report.html || echo "ZAP Scan failed"
                                    ''', returnStatus: true

                                echo "Report a file Zap"

                                archiveArtifacts artifacts: 'zap_report.html', fingerprint: true

                                if (fileExists('zap_report.html')) {
                                    echo "ZAP Scan completed successfully, report generated."
                                } else {
                                    echo "ZAP Scan failed to generate report."
                                    currentBuild.result = 'UNSTABLE'
                                }




                            }
                        }


                        }
        }
        stage('Publish') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'nexus', passwordVariable: 'NEXUS_PASSWORD', usernameVariable: 'NEXUS_USERNAME')]) {
                        sh """
                        docker login ${NEXUS_DOCKER_PUSH_INDEX} -u $NEXUS_USERNAME -p $NEXUS_PASSWORD
                        docker tag ${DOCKER_IMAGE_NAME}:${BUILD_NUMBER} ${NEXUS_DOCKER_PUSH_INDEX}/${NEXUS_DOCKER_PUSH_PATH}/${DOCKER_IMAGE_NAME}:latest
                        docker push ${NEXUS_DOCKER_PUSH_INDEX}/${NEXUS_DOCKER_PUSH_PATH}/${DOCKER_IMAGE_NAME}:latest
                        """
                    }
                }
            }
        }
        stage('Deploy') {
            agent { label 'dsb-node-01' }
            steps {
                script {
                        echo 'Deploying to DSB Node 01'
                        // Port 3000 is already in use, use 6000 for this application
                        sh '''
                        docker pull ${NEXUS_DOCKER_PUSH_INDEX}/${NEXUS_DOCKER_PUSH_PATH}/${DOCKER_IMAGE_NAME}:latest
                        docker stop ${DOCKER_IMAGE_NAME} || true
                        docker rm ${DOCKER_IMAGE_NAME} || true
                        docker run -d --name ${DOCKER_IMAGE_NAME} -p 8084:3000 ${NEXUS_DOCKER_PUSH_INDEX}/${NEXUS_DOCKER_PUSH_PATH}/${DOCKER_IMAGE_NAME}:latest
                        '''
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