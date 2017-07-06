pipeline {
    agent any
    environment {
            // You need to specify 4 required environment variables first, they are going to be used for the following IBM Cloud DevOps steps
            IBM_CLOUD_DEVOPS_CREDS = credentials('BM_CRED')
            IBM_CLOUD_DEVOPS_ORG = 'fuse@jp.ibm.com'
            IBM_CLOUD_DEVOPS_APP_NAME = 'WheatherApp-20170525GVT'
            IBM_CLOUD_DEVOPS_TOOLCHAIN_ID = '6bc113bb-5331-493c-8da1-45631fc33adf'
            IBM_CLOUD_DEVOPS_WEBHOOK_URL = 'https://jenkins:f761d3a3-b620-45a6-9e15-d4a52a4eb211:b8e4b101-cacd-4997-bdac-1a62047f7cec@devops-api-integration.stage1.ng.bluemix.net/v1/toolint/messaging/webhook/publish'
        }
    stages {
        stage('SCM') {
            steps {
                git 'https://github.com/mfuse/devopsinsights-toolchain-20170525GVT.git'
            }
        }
        stage ('SonarQube analysis') {
            steps {
                script {
                    // requires SonarQube Scanner 2.8+
                    def scannerHome = tool 'SQ Scanner GVT';
                   
                    withSonarQubeEnv('SonarQube GVT') {

                        env.SQ_HOSTNAME = SONAR_HOST_URL;
                        env.SQ_AUTHENTICATION_TOKEN = SONAR_AUTH_TOKEN;
                        env.SQ_PROJECT_KEY = "devopsinsights-toolchain-20170525GVT";

                      sh "${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=${SQ_PROJECT_KEY} \
                                -Dsonar.sources=.";
                               // -Dsonar.organization=";
                    }
                }
            }
        }
        stage ("SonarQube Quality Gate") {
             steps {
                script {

                    def qualitygate = waitForQualityGate()
                    if (qualitygate.status != "OK") {
                        error "Pipeline aborted due to quality gate coverage failure: ${qualitygate.status}"
                    }
                }
             }
             post {
                always {
                    publishSQResults SQHostURL: "${SQ_HOSTNAME}", SQAuthToken: "${SQ_AUTHENTICATION_TOKEN}", SQProjectKey:"${SQ_PROJECT_KEY}"
                }
             }
        }
    }
}
