#!groovy
/*
    This is an sample Jenkins file for the Weather App, which is a node.js application that has unit test, code coverage
    and functional verification tests, deploy to staging and production environment and use IBM Cloud DevOps gate.
    We use this as an example to use our plugin in the Jenkinsfile
    Basically, you need to specify required 4 environment variables and then you will be able to use the 4 different methods
    for the build/test/deploy stage and the gate
 */
pipeline {
    agent any
    environment {
        // You need to specify 4 required environment variables first, they are going to be used for the following IBM Cloud DevOps steps
        IBM_CLOUD_DEVOPS_CREDS = credentials('BM_CRED')
        IBM_CLOUD_DEVOPS_API_KEY = credentials('API_KEY')
        IBM_CLOUD_DEVOPS_ORG = 'fuse@jp.ibm.com'
        IBM_CLOUD_DEVOPS_APP_NAME = 'WheatherApp-20170525GVT'
        IBM_CLOUD_DEVOPS_TOOLCHAIN_ID = '6bc113bb-5331-493c-8da1-45631fc33adf'
        IBM_CLOUD_DEVOPS_WEBHOOK_URL = 'https://jenkins:f761d3a3-b620-45a6-9e15-d4a52a4eb211:b8e4b101-cacd-4997-bdac-1a62047f7cec@devops-api-integration.stage1.ng.bluemix.net/v1/toolint/messaging/webhook/publish'
    }
    tools {
        nodejs 'recent'
    }
    stages {
        stage('ビルド') {
            environment {
                // get git commit from Jenkins
                GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
                GIT_BRANCH = 'master'
                GIT_REPO = "https://github.com/mfuse/devopsinsights-toolchain-20170525GVT"
            }
            steps {
                checkout scm
                sh 'npm --version'
                sh 'npm install'
                sh 'grunt dev-setup --no-color'
            }
            // post build section to use "publishBuildRecord" method to publish build record
            post {
                success {
                    publishBuildRecord gitBranch: "${GIT_BRANCH}", gitCommit: "${GIT_COMMIT}", gitRepo: "${GIT_REPO}", result:"SUCCESS"
                }
                failure {
                    publishBuildRecord gitBranch: "${GIT_BRANCH}", gitCommit: "${GIT_COMMIT}", gitRepo: "${GIT_REPO}", result:"FAIL"
                }
            }
        }
        stage('単体テストとコードカバレッジ') {
            steps {
                sh 'grunt dev-test-cov --no-color -f'
            }
            // post build section to use "publishTestResult" method to publish test result
            post {
                always {
                    publishTestResult type:'unittest', fileLocation: './mochatest.json'
                    publishTestResult type:'code', fileLocation: './tests/coverage/reports/coverage-summary.json'
                }
            }
        }
               stage('SCM') {
            steps {
                git 'https://github.com/mfuse/devopsinsights-toolchain-20170525GVT.git'
            }
        }
        stage ('SonarQube analysis') {
            steps {
                script {
                    // requires SonarQube Scanner 2.8+
                    def scannerHome = tool 'SonarQube Scanner GVT';
                   
                    withSonarQubeEnv('SonarQube GVT') {

                        env.SQ_HOSTNAME = SONAR_HOST_URL;
                        env.SQ_AUTHENTICATION_TOKEN = SONAR_AUTH_TOKEN;
                      //  env.SQ_HOSTNAME = "http://bmxcd.dhcp.hakozaki.ibm.com:9000";
                      //  env.SQ_AUTHENTICATION_TOKEN = "4e838a5fed188b8cb5ff01c3b1dc80d416d786b8";
                        env.SQ_PROJECT_KEY = "devopsinsights-toolchain-20170525GVT";
                       

                      sh "${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=${SQ_PROJECT_KEY} \
                                -Dsonar.sources=.";
                         //       -Dsonar.organization=default-organization";
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
        stage('ステージングにデプロイ') {
            steps {
                // Push the Weather App to Bluemix, staging space
                sh '''
                        echo "CF ログイン..."
                        cf api https://api.stage1.ng.bluemix.net
                     #  cf login -u $IBM_CLOUD_DEVOPS_CREDS_USR -p $IBM_CLOUD_DEVOPS_CREDS_PSW -o $IBM_CLOUD_DEVOPS_ORG -s ステージング
                        cf login -u apikey -p $IBM_CLOUD_DEVOPS_API_KEY -o $IBM_CLOUD_DEVOPS_ORG -s ステージング
                        
                        echo "デプロイ中...."
                        export CF_APP_NAME="staging-$IBM_CLOUD_DEVOPS_APP_NAME"
                        cf delete $CF_APP_NAME -f
                        cf push $CF_APP_NAME -n $CF_APP_NAME -m 64M -i 1

                        # use "cf icd --create-connection" to enable traceability
                        cf icd --create-connection $IBM_CLOUD_DEVOPS_WEBHOOK_URL $CF_APP_NAME

                        export APP_URL=http://$(cf app $CF_APP_NAME | grep urls: | awk '{print $2}')
                    '''
            }
            // post build section to use "publishDeployRecord" method to publish deploy record and notify OTC of stage status
            post {
                success {
                    publishDeployRecord environment: "STAGING", appUrl: "http://staging-${IBM_CLOUD_DEVOPS_APP_NAME}.mybluemix.net", result:"SUCCESS"
                    // use "notifyOTC" method to notify otc of stage status
                    notifyOTC stageName: "ステージングにデプロイ", status: "SUCCESS"
                }
                failure {
                    publishDeployRecord environment: "STAGING", appUrl: "http://staging-${IBM_CLOUD_DEVOPS_APP_NAME}.mybluemix.net", result:"FAIL"
                    // use "notifyOTC" method to notify otc of stage status
                    notifyOTC stageName: "ステージングにデプロイ", status: "FAIL"
                }
            }
        }
        stage('FVT') {
            //set the APP_URL as the environment variable for the fvt
            environment {
                APP_URL = "http://staging-${IBM_CLOUD_DEVOPS_APP_NAME}.mybluemix.net"
            }
            steps {
                sh 'grunt fvt-test --no-color -f'
            }
            // post build section to use "publishTestResult" method to publish test result
            post {
                always {
                    publishTestResult type:'fvt', fileLocation: './mochafvt.json', environment: 'STAGING'
                }
            }
        }
        stage('ゲート') {
            steps {
                // use "evaluateGate" method to leverage IBM Cloud DevOps gate
          //      evaluateGate policy: 'Weather App Policy', forceDecision: 'true'
          evaluateGate policy: '天気予報アプリポリシー①', forceDecision: 'true'
            }
        }
        stage('実稼働にデプロイ') {
            steps {
                // Push the Weather App to Bluemix, production space
                sh '''
                        echo "CF ログイン..."
                        cf api https://api.stage1.ng.bluemix.net
                      # cf login -u $IBM_CLOUD_DEVOPS_CREDS_USR -p $IBM_CLOUD_DEVOPS_CREDS_PSW -o $IBM_CLOUD_DEVOPS_ORG -s 実稼働
                        cf login -u apikey -p $IBM_CLOUD_DEVOPS_API_KEY -o $IBM_CLOUD_DEVOPS_ORG -s 実稼働

                        echo "デプロイ中...."
                        export CF_APP_NAME="prod-$IBM_CLOUD_DEVOPS_APP_NAME"
                        cf delete $CF_APP_NAME -f
                        cf push $CF_APP_NAME -n $CF_APP_NAME -m 64M -i 1

                        # use "cf icd --create-connection" to enable traceability
                        cf icd --create-connection $IBM_CLOUD_DEVOPS_WEBHOOK_URL $CF_APP_NAME

                        export APP_URL=http://$(cf app $CF_APP_NAME | grep urls: | awk '{print $2}')
                    '''
            }
            // post build section to use "publishDeployRecord" method to publish deploy record and notify OTC of stage status
            post {
                success {
                    publishDeployRecord environment: "PRODUCTION", appUrl: "http://prod-${IBM_CLOUD_DEVOPS_APP_NAME}.mybluemix.net", result:"SUCCESS"
                    // use "notifyOTC" method to notify otc of stage status
                    notifyOTC stageName: "実稼働にデプロイ", status: "SUCCESS"
                }
                failure {
                    publishDeployRecord environment: "PRODUCTION", appUrl: "http://prod-${IBM_CLOUD_DEVOPS_APP_NAME}.mybluemix.net", result:"FAIL"
                    // use "notifyOTC" method to notify otc of stage status
                    notifyOTC stageName: "実稼働にデプロイ", status: "FAIL"
                }
            }
        }
    }
}
