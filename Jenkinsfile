pipeline {
    agent any

    environment {
        AWS_REGION = 'eu-central-1'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build, Scan & Push All Services') {
            steps {
                withCredentials([string(credentialsId: 'aws-account-id', variable: 'AWS_ACCOUNT_ID')]) {
                    script {
                        env.IMAGE_TAG    = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        env.ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                    }
                    sh '''
                        aws ecr get-login-password --region ${AWS_REGION} \
                            | docker login --username AWS --password-stdin ${ECR_REGISTRY}

                        build_scan_push() {
                            SERVICE=$1
                            BUILD_PATH=$2
                            ECR_IMAGE="${ECR_REGISTRY}/cloudcommerce/${SERVICE}:${IMAGE_TAG}"

                            echo "========== Building ${SERVICE} =========="
                            docker build -t ${ECR_IMAGE} ${BUILD_PATH}

                            echo "========== Scanning ${SERVICE} =========="
                            trivy image --exit-code 0 --severity HIGH,CRITICAL --no-progress ${ECR_IMAGE}

                            echo "========== Pushing ${SERVICE} =========="
                            docker push ${ECR_IMAGE}

                            docker rmi ${ECR_IMAGE} || true
                        }

                        build_scan_push adservice                src/adservice
                        build_scan_push cartservice              src/cartservice/src
                        build_scan_push checkoutservice          src/checkoutservice
                        build_scan_push currencyservice          src/currencyservice
                        build_scan_push emailservice             src/emailservice
                        build_scan_push frontend                 src/frontend
                        build_scan_push loadgenerator            src/loadgenerator
                        build_scan_push paymentservice           src/paymentservice
                        build_scan_push productcatalogservice    src/productcatalogservice
                        build_scan_push recommendationservice    src/recommendationservice
                        build_scan_push shippingservice          src/shippingservice
                        build_scan_push shoppingassistantservice src/shoppingassistantservice
                    '''
                }
            }
        }

        stage('Update values.yaml') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-account-id', variable: 'AWS_ACCOUNT_ID'),
                    usernamePassword(credentialsId: 'github-token', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')
                ]) {
                    sh '''
                        git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/Dennis4507/cloudcommerce-devops.git infra
                        cd infra

                        git config user.email "jenkins@cloudcommerce.dev"
                        git config user.name "Jenkins"

                        NEW_REPO="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/cloudcommerce"
                        sed -i "s|  repository:.*|  repository: ${NEW_REPO}|" kubernetes/apps/online-boutique/values.yaml
                        sed -i "s|  tag:.*|  tag: \\"${IMAGE_TAG}\\"|" kubernetes/apps/online-boutique/values.yaml

                        git add kubernetes/apps/online-boutique/values.yaml
                        git commit -m "ci: update all services to ${IMAGE_TAG} [skip ci]"
                        git push
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'rm -rf infra'
        }
    }
}
