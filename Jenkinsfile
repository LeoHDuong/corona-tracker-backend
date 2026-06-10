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

	stage('Build JAR') {
	    steps {
		sh 'mvn clean package -DskipTests'
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
