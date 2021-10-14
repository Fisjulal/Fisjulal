pipeline {
    agent { 
        label 'OCI-agent' 
    } 
    options {
        // Disable concurrent builds for a same branch
        disableConcurrentBuilds()        
        // Abort job if it runs more than 1 hour
        timeout(time: 1, unit: 'HOURS') 
    }
    environment {
        // vMM.mm.pp-rc.NN
        TAG_PATTERN_TEST = '^v\\d{1,2}?\\.\\d{1,2}?\\.\\d{1,2}?-rc\\.\\d{1,2}?\$'
        // vMM.mm.pp
        TAG_PATTERN_PROD = 'v\\d{1,2}?\\.\\d{1,2}?\\.\\d{1,2}?\$'        
        // Get committer name
        GIT_COMMITTER_NAME = sh (
            script: 'git log -1 --format=\'%cn\' ${GIT_COMMIT}',
            returnStdout: true
        ).trim()
        // Get current branch, used for detecting tag on the master branch
        BRANCH_NAME_REAL = sh (
            script: 'git branch -a --contains ${GIT_COMMIT}',
            returnStdout: true
        ).trim()
		// Get last commiter mail
        GIT_COMMITER_EMAIL = sh (
            script: 'git --no-pager show -s --format=\'%ae\'',
            returnStdout: true
        ).trim()     
    }  

    stages {
        stage('Prepare') {
            steps {
                // Send start message to devops channel
                slackSend channel: '#devops', color: 'warning', message: "${env.JOB_NAME} #${env.BUILD_NUMBER} - Started by ${env.GIT_COMMITTER_NAME}. (<${env.BUILD_URL}|Open>)"
            }    
        }
        stage('Build') {            
            when { branch 'master' }
            steps {
                sh '''                    
                    set +ex
                    ## Set Java version
                    source "/home/opc/.sdkman/bin/sdkman-init.sh"
                    sdk use java jdk1.8.0_151
                    set -ex
                    
                    ## Print current java version
                    java -version
                    
                    ./gradlew clean bootWar
                '''
            }
        } 
        stage('Test') {
            when { branch 'master' }
            steps {
                sh '''
                    set +ex
                    ## Set Java version
                    source "/home/opc/.sdkman/bin/sdkman-init.sh"
                    sdk use java jdk1.8.0_151
                    set -ex
                    
                    ## Print current java version
                    java -version
                    
                    ./gradlew test
                '''
            }
        }
        stage('SonarQube') {
            when { branch 'master' }
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''                        
                        set +ex
                        ## Set Java version
                        source "/home/opc/.sdkman/bin/sdkman-init.sh"
                        sdk use java jdk1.8.0_151
                        set -ex
                        
                        ## Print current java version
                        java -version
                        
                        ./gradlew sonarqube \
                            -Dsonar.projectName=ALIMC-FISCALIZATION-SERVICE \
                            -Dsonar.projectKey=ALIMC-FISCALIZATION-SERVICE
                    '''
                }
            }
        }
        stage('Deploy to DEV') {
            when { branch 'master' }
            steps {
                dir('resources/deploy') {
                    sh (returnStatus: true, script: '''
                        set +ex
                        ## Weblogic environment
                        . /oracle/wls12213/wlserver/server/bin/setWLSEnv.sh
                        set -ex
                        
                        ./undeploy_dev.sh
                    ''')
                    sh '''
                        set +ex
                        ## Weblogic environment
                        . /oracle/wls12213/wlserver/server/bin/setWLSEnv.sh
                        set -ex
                        
                        ./deploy_dev.sh
                    '''
                }
            }
        }    
        stage('Deploy to TEST') {
            when { branch 'master' }
            //when { tag pattern: TAG_PATTERN_TEST, comparator: "REGEXP" }
            steps {
                dir('resources/deploy') {
                    sh (returnStatus: true, script: '''
                        set +ex
                        ## Weblogic environment
                        . /oracle/wls12213/wlserver/server/bin/setWLSEnv.sh
                        set -ex
                        
                        ./undeploy_test-neos.sh
                    ''')
                    sh '''
                        set +ex
                        ## Weblogic environment
                        . /oracle/wls12213/wlserver/server/bin/setWLSEnv.sh
                        set -ex
                        
                        ./deploy_test-neos.sh
                    '''
                }
            }
        } 
    }
        
    post {
        always {  
            echo 'PRINTENV' 
            sh 'printenv'               
        }  
        /*
        success { 
        }  
        unstable {  
        }  
        changed {
        } */
        failure {
			echo 'POST failure'  
            slackSend (color: '#ff595e', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL}), commiter: ${env.GIT_COMMITER_EMAIL}")
            emailext to: "${env.GIT_COMMITER_EMAIL}", subject: "Jenkins error: ", body: "Failed project ${env.JOB_NAME} with build number ${env.BUILD_NUMBER}.<br/> Build URL: ${env.BUILD_URL}", mimeType:'text/html';  
        }
    }    
}
