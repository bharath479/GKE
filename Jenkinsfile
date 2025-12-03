pipeline {
    agent any
    
    environment {
        PROJECT_ID = "bilvantisaimlproject"
        CLUSTER = "devops-gke"
        REGION = "us-central1"
        REPO_URL = "https://github.com/bharath479/GKE.git"
        USE_GKE_GCLOUD_AUTH_PLUGIN = "True"
    }

    stages {

        /* ----------------------
           Stage 1: Clone Repository
        ----------------------- */
        stage('Clone Repository') {
            steps {
                sh '''
                    rm -rf repo
                    git clone $REPO_URL repo
                    echo "Repo cloned"
                    ls -ltr repo
                '''
            }
        }

        /* ----------------------
           Stage 2: Authenticate to GKE
        ----------------------- */
        stage('Authenticate to GKE') {
            steps {
                withCredentials([file(credentialsId: 'GCP-KEY', variable: 'GCLOUD_KEY')]) {
                    sh '''
                        echo "Authenticating to GCP..."
                        gcloud auth activate-service-account --key-file=$GCLOUD_KEY
                        gcloud config set project $PROJECT_ID
                        gcloud config set compute/region $REGION
                        echo "Fetching GKE Credentials..."
                        gcloud container clusters get-credentials $CLUSTER --region $REGION --project $PROJECT_ID
                    '''
                }
            }
        }

        /* ----------------------
           Stage 3: Detect Current Version from svc.yaml
        ----------------------- */
        stage('Detect Current Active Version') {
            steps {
                script {
                    echo "Reading svc.yaml to detect active version..."

                    def version = sh(
                        script: "grep 'version:' repo/svc.yaml | awk '{print \$2}'",
                        returnStdout: true
                    ).trim()

                    echo "Current Version in svc.yaml = ${version}"

                    if (version == "blue") {
                        env.NEW_VERSION = "blue"
                        env.CURRENT_VERSION = "green"
                    } else if (version == "green") {
                        env.NEW_VERSION = "green"
                        env.CURRENT_VERSION = "blue"
                    } else {
                        error "svc.yaml does NOT contain version: blue or version: green"
                    }
                }
            }
        }

        /* ----------------------
           Stage 4: Deploy HTML ConfigMap for NEW Version
        ----------------------- */
        stage('Deploy HTML ConfigMap') {
            steps {
                script {
                    echo "Deploying ConfigMap for ${env.NEW_VERSION}"

                    sh """
                        cd repo
                        kubectl apply -f ${env.NEW_VERSION}-html-configmap.yaml
                    """
                }
            }
        }

        /* ----------------------
           Stage 5: Deploy NEW Version Deployment
        ----------------------- */
        stage('Deploy New Version') {
            steps {
                script {
                    echo "Deploying NEW version: ${env.NEW_VERSION}"

                    sh """
                        cd repo
                        kubectl apply -f ${env.NEW_VERSION}-deployment.yaml
                    """
                }
            }
        }

        /* ----------------------
           Stage 6: Health Check NEW Version
        ----------------------- */
        stage('Health Check New Version') {
            steps {
                script {
                    echo "Checking rollout of NEW version: ${env.NEW_VERSION}"

                    sh """
                        kubectl rollout status deployment/nginx-${env.NEW_VERSION} --timeout=180s
                    """
                }
            }
        }

        /* ----------------------
           Stage 7: Switch Traffic to NEW Version
        ----------------------- */
        stage('Switch Traffic') {
            steps {
                script {
                    echo "Switching service traffic to ${env.NEW_VERSION}..."

                    sh """
                        kubectl patch service nginx-service -p '{"spec":{"selector":{"app":"nginx","version":"${env.NEW_VERSION}"}}}'
                    """
                }
            }
        }

        /* ----------------------
           Stage 8: Delete OLD Version
        ----------------------- */
        stage('Delete Old Version') {
            steps {
                script {
                    echo "Deleting OLD version: ${env.CURRENT_VERSION}"

                    sh """
                        kubectl delete deployment nginx-${env.CURRENT_VERSION} --ignore-not-found=true
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Blue-Green Deployment (with custom HTML) completed successfully!"
        }
        failure {
            echo "Deployment FAILED! Manual rollback may be required."
        }
    }
}
