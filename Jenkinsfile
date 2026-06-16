pipeline {
    agent {
        label 'docker-agent'
    }

    environment {
        HARBOR_REGISTRY = '10.178.57.127'
        HARBOR_PROJECT = 'corona-tracker'
        IMAGE_NAME = 'backend'
        IMAGE_TAG = "${BUILD_NUMBER}"
        HELM_REPO = 'git@github.com:LeoHDuong/corona-tracker-helm.git'
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
                    gitleaks detect --source . --report-format json --report-path gitleaks-report.json --exit-code 0
                '''
                archiveArtifacts artifacts: 'gitleaks-report.json', allowEmptyArchive: true
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
	            withCredentials([string(credentialsId: 'vault-token', variable: 'VAULT_TOKEN')]) {
	                sh """
	                    export VAULT_ADDR=http://nexus:8200
	                    export VAULT_TOKEN=\${VAULT_TOKEN}
	                    set +x
	                    SONAR_TOKEN=\$(vault kv get -field=token secret/corona-tracker/sonarqube)
	                    set -x
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
                    docker build -t ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG} \
                        ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Push to Harbor') {
	    steps {
	        withCredentials([string(credentialsId: 'vault-token', variable: 'VAULT_TOKEN')]) {
	            sh """
	                export VAULT_ADDR=http://nexus:8200
	                export VAULT_TOKEN=\${VAULT_TOKEN}
	                set +x
	                HARBOR_USER=\$(vault kv get -field=username secret/corona-tracker/harbor)
	                HARBOR_PASS=\$(vault kv get -field=password secret/corona-tracker/harbor)
	                docker login ${HARBOR_REGISTRY} -u \$HARBOR_USER -p \$HARBOR_PASS 2>/dev/null
	                set -x
	                docker push ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}
	                docker push ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:latest
	            """
	        }
	    }
	}

	stage('Sign Image - Cosign') {
	    steps {
	        withCredentials([string(credentialsId: 'vault-token', variable: 'VAULT_TOKEN')]) {
	            sh """
	                export VAULT_ADDR=http://nexus:8200
	                export VAULT_TOKEN=\${VAULT_TOKEN}
	                set +x
	                COSIGN_PASSWORD=\$(vault kv get -field=password secret/corona-tracker/cosign)
	                export COSIGN_PASSWORD
	                set -x
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
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'github-ssh',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh """
                        GIT_SSH_COMMAND="ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no" \
                        git clone ${HELM_REPO} helm-charts
                        helm upgrade --install backend helm-charts/backend \
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
