pipeline {
    agent {
        label 'jenkins-worker'
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
        	sh '''
            	    export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
            	    export PATH=$JAVA_HOME/bin:$PATH
            	    echo "JAVA_HOME is: $JAVA_HOME"
            	    java -version
            	    mvn -version
            	    mvn clean package -DskipTests
        	'''
     	    }
    	}

	stage('SonarQube Analysis') {
	    steps {
	        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
	            sh """
	                sonar-scanner \
	                    -Dsonar.projectKey=corona-tracker-backend \
	                    -Dsonar.sources=src \
	                    -Dsonar.java.binaries=target/classes \
	                    -Dsonar.host.url=http://sonarqube:9000 \
	                    -Dsonar.token=\${SONAR_TOKEN}
	            """
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
                withCredentials([usernamePassword(
                    credentialsId: 'harbor-credentials',
                    usernameVariable: 'HARBOR_USER',
                    passwordVariable: 'HARBOR_PASS'
                )]) {
                    sh """
                        docker login ${HARBOR_REGISTRY} -u ${HARBOR_USER} -p ${HARBOR_PASS}
                        docker push ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${IMAGE_NAME}:latest
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
                            --kubeconfig /home/leo/.kube/config
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
