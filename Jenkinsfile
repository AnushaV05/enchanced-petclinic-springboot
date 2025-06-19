pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        IMAGE_NAME        = "springbootapp"
        IMAGE_TAG         = "${BUILD_NUMBER}"
        TENANT_ID         = "ed56eaf4-7b02-4642-a7a8-f3e7a7b78ef7"
        ACR_NAME          = "jeevanacr20250619"
        ACR_LOGIN_SERVER  = "${ACR_NAME}.azurecr.io"
        FULL_IMAGE_NAME   = "${ACR_LOGIN_SERVER}/${IMAGE_NAME}:${IMAGE_TAG}"
        RESOURCE_GROUP    = "myResourceGroup"
        AKS_CLUSTER       = "springboot"
        K8S_NAMESPACE     = "default"
        K8S_DEPLOYMENT    = "springboot-app"
    }

    stages {
        stage('Checkout From Git') {
            steps {
                git branch: 'prod', url: 'https://github.com/AnushaV05/enchanced-petclinic-springboot.git'
            }
        }

        stage('Maven Compile') {
            steps {
                echo "This is Maven Compile Stage"
                sh 'mvn compile'
            }
        }

        stage('Maven Test') {
            steps {
                echo "This is Maven Test Stage"
                sh 'mvn test'
            }
        }

        stage('File System Scan By Trivy') {
            steps {
                echo "Trivy Scan Started"
                sh 'trivy fs --format table --output trivy-report.txt --severity HIGH,CRITICAL .'
            }
        }

        stage('Maven Package') {
            steps {
                echo "Maven Package Started"
                sh 'mvn package'
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    echo "Docker Build Started"
                    sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."

                }
            }
        }

        stage('Azure Login to ACR') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'azure-acr-sp', usernameVariable: 'AZURE_USERNAME', passwordVariable: 'AZURE_PASSWORD')]) {
                    script {
                        echo "Azure Login Started"
                        sh '''
                            az login --service-principal -u $AZURE_USERNAME -p $AZURE_PASSWORD --tenant $TENANT_ID
                            az acr login --name $ACR_NAME
                        '''
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
  steps {
    sh '''
      export TRIVY_CACHE_DIR=/tmp/trivy-cache
      mkdir -p $TRIVY_CACHE_DIR
      trivy image --cache-dir $TRIVY_CACHE_DIR --format table --output trivy-report.txt --severity HIGH,CRITICAL springbootapp:35
    '''
  }
}


            }
        }

        stage('Docker Push to ACR') {
            steps {
                script {
                    echo "Docker Push Started"
                    sh '''
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${FULL_IMAGE_NAME}
                        docker push ${FULL_IMAGE_NAME}
                    '''
                }
            }
        }

        stage('Azure Login to Kubernetes') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'azure-acr-sp', usernameVariable: 'AZURE_USERNAME', passwordVariable: 'AZURE_PASSWORD')]) {
                    script {
                        echo "Azure Login to Kubernetes Started"
                        sh '''
                            az login --service-principal -u $AZURE_USERNAME -p $AZURE_PASSWORD --tenant $TENANT_ID
                            az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER --overwrite-existing    
                        '''
                    }
                }
            }
        }

        stage('Kubernetes Deployment') {
            steps {
                script {
                    echo "Kubernetes Deployment Stage Started"

                    def output = sh(
                        script: "kubectl get deployment ${K8S_DEPLOYMENT} -n $K8S_NAMESPACE --ignore-not-found",
                        returnStdout: true
                    ).trim()

                    def deploymentExists = output != ""

                    if (deploymentExists) {
                        echo "Deployment exists. Performing rolling update with new image: ${BUILD_NUMBER}"
                        sh """
                            kubectl set image deployment/${K8S_DEPLOYMENT} \
                            ${K8S_DEPLOYMENT}=${FULL_IMAGE_NAME} \
                            -n $K8S_NAMESPACE
                        """
                    } else {
                        echo "Deployment not found. Creating new deployment from template"
                        sh """
                            sed "s/__IMAGE_TAG__/${BUILD_NUMBER}/" k8s/sprinboot-deployment.yaml > k8s/tmp-deployment.yaml
                            kubectl apply -f k8s/tmp-deployment.yaml -n $K8S_NAMESPACE
                        """
                    }
                }
            }
        }

        stage('Check Deployment') {
            steps {
                sh '''
                    echo "Checking deployment status..."
                    kubectl rollout status deployment/${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE}
                '''
            }
        }
    }
}
