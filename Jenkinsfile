pipeline {
    agent {
        label 'docker-agent'
    }

    environment {
        HARBOR_REGISTRY = '10.178.57.127'
        HARBOR_PROJECT = 'corona-tracker'
        IMAGE_NAME = 'backend'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Secret Scan - Gitleaks') {
            steps {
                sh '''
                    gitleaks detect --source . --report-format json \
                        --report-path gitleaks-report.json --exit-code 0
                '''
                archiveArtifacts artifacts: 'gitleaks-report.json', allowEmptyArchive: true
            }
        }

	stage('Semgrep') {
	    steps {
	        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
	            sh '''
	                semgrep scan \
	                    --config=auto \
	                    --json \
	                    --output=semgrep-report.json \
	                    src/
	            '''
	            archiveArtifacts artifacts: 'semgrep-report.json', allowEmptyArchive: true
	        }
	    }
	}

        stage('Build JAR') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withVault(vaultSecrets: [[
                        path: 'secret/corona-tracker/sonarqube',
                        secretValues: [[vaultKey: 'token', envVar: 'SONAR_TOKEN']]
                    ]]) {
                        sh """
                            sonar-scanner \
                                -Dsonar.projectKey=corona-tracker-backend \
                                -Dsonar.sources=src \
                                -Dsonar.java.binaries=target/classes \
                                -Dsonar.host.url=http://sonarqube:9000 \
                                -Dsonar.token=\$SONAR_TOKEN
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build \
                        --memory=512m \
                        --memory-swap=1g \
                        -t ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG} \
                        ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Push to Harbor') {
            steps {
                withVault(vaultSecrets: [[
                    path: 'secret/corona-tracker/harbor',
                    secretValues: [
                        [vaultKey: 'username', envVar: 'HARBOR_USER'],
                        [vaultKey: 'password', envVar: 'HARBOR_PASS']
                    ]
                ]]) {
                    sh """
                        docker login ${HARBOR_REGISTRY} -u \$HARBOR_USER -p \$HARBOR_PASS 2>/dev/null
                        docker push ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('Sign Image - Cosign') {
            steps {
                withVault(vaultSecrets: [[
                    path: 'secret/corona-tracker/cosign',
                    secretValues: [[vaultKey: 'password', envVar: 'COSIGN_PASSWORD']]
                ]]) {
                    sh """
                        cosign sign --key /etc/cosign/cosign.key \
                            -a "pipeline=jenkins" \
                            -a "build=${IMAGE_TAG}" \
                            --tlog-upload=false \
                            ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Deploy with Helm') {
            steps {
                withVault(vaultSecrets: [[
                    path: 'secret/corona-tracker/nexus',
                    secretValues: [
                        [vaultKey: 'username', envVar: 'NEXUS_USER'],
                        [vaultKey: 'password', envVar: 'NEXUS_PASS']
                    ]
                ]]) {
                    sh """
                        helm repo add nexus-helm http://nexus:8081/repository/helm-charts/ \
                            --username \$NEXUS_USER --password \$NEXUS_PASS
                        helm repo update

                        helm template backend nexus-helm/webapp \
                            --version 0.2.0 \
                            -f values.yaml \
                            --set image.repository=${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME} \
                            --set image.tag=${IMAGE_TAG}

                        helm upgrade --install backend nexus-helm/webapp \
                            --version 0.2.0 \
                            -f values.yaml \
                            --set image.repository=${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME} \
                            --set image.tag=${IMAGE_TAG} \
                            --kubeconfig /home/jenkins/.kube/config
                    """
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout ${HARBOR_REGISTRY} || true'
            cleanWs()
        }
        success {
            echo "Backend deployed successfully! Image tag: ${IMAGE_TAG}"
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}
