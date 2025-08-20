pipeline {
    agent  {
        label 'AGENT-1'
    }
    environment { 
        appVersion = ''
        REGION = "us-east-1"
        ACC_ID = "169189304039"
        PROJECT = "roboshop"
        COMPONENT = "catalogue"
    }
    options {
        timeout(time: 30, unit: 'MINUTES') 
        disableConcurrentBuilds()
    }
    parameters {
        string(name: 'appVersion', description: 'Image version of the application')
        choice(name: 'deploy_to', choices: ['dev', 'qa', 'prod'], description: 'Pick the Environment')
    }
    // Build
    stages {
        stage('Deploy') {
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                        sh """
                            aws eks update-kubeconfig --region $REGION --name "$PROJECT-${params.deploy_to}"
                            kubectl get nodes
                            kubectl apply -f 01-namespace.yaml
                            sed -i "s/IMAGE_VERSION/${params.appVersion}/g" values-${params.deploy_to}.yaml
                            helm upgrade --install $COMPONENT -f values-${params.deploy_to}.yaml -n $PROJECT .
                        """
                    }
                }
            }
        }
        stage('Check Status'){
            steps{
                script{
                    withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                def deploymentStatus = sh(returnStdout: true, script: "kubectl rollout status deployment/catalogue --timeout=30s -n $PROJECT || echo FAILED").trim()
                
                // rollout success does not mean pods are healthy
                def podStatus = sh(returnStdout: true, script: "kubectl get pods -n $PROJECT -l component=${COMPONENT} -o jsonpath='{.items[*].status.containerStatuses[*].state.waiting.reason}' || true").trim()

                if (deploymentStatus.contains("successfully rolled out") && podStatus == "") {
                    echo "Deployment is success ✅"
                } else {
                    echo "Deployment failed ❌"
                    sh """
                        echo "Rolling back..."
                        helm rollback $COMPONENT -n $PROJECT
                        sleep 20
                    """
                    def rollbackStatus = sh(returnStdout: true, script: "kubectl rollout status deployment/catalogue --timeout=30s -n $PROJECT || echo FAILED").trim()
                    if (rollbackStatus.contains("successfully rolled out")) {
                        error "Deployment failed, Rollback succeeded"
                    }
                    else {
                        error "Deployment failed, Rollback also failed. App is down."
                    }
                }
            }
        }
    }
}
        
    }

    post { 
        always { 
            echo 'I will always say Hello again!'
            deleteDir()
        }
        success { 
            echo 'Hello Success'
        }
        failure { 
            echo 'Hello Failure'
        }
    }
}