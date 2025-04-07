pipeline {
    agent any

    environment {
        target_name = "http://172.17.0.1:3000"
    }

    stages {
        stage('Clone') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/master']],
                    extensions: [],
                    userRemoteConfigs: [[
                        credentialsId: 'jenkins-user-github',
                        url: 'https://github.com/markyboi246/Test'
                    ]]
                )
            }
        }

        stage('Build & Serve') {
            steps {
                sh '''
                    echo "Dependicies"
                    npm install

                    echo "Starting"
                    npm start &

                    echo "Waiting for app"
                    sleep 20
                '''
            }
        }

        stage('Security Scan') {
            steps {
                script {
                    echo "Running ZAP Scan"

                    sh '''
                        docker run --rm -t --network=host \
                        ghcr.io/zaproxy/zaproxy:stable zap-full-scan.py \
                        -t ${target_name} \
                        -r zap_report.html || echo "ZAP Scan failed"
                    '''

                    echo "Checking for ZAP report..."

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
}


