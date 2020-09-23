node {
    environment {
        APP_NAME = "SampleNodeJs"
        STAGING = "Staging"
        PRODUCTION = "Production"
    }
 
    stage('Checkout') {
        // Checkout our application source code
        git url: 'https://github.com/nikhilgoenkatech/JenkinsBankApp.git'
        
        // into a dynatrace-cli subdirectory we checkout the CLI
        dir ('dynatrace-cli') {
            git url: 'https://github.com/Dynatrace/dynatrace-cli.git'
        }
    }

    stage('Build') {
        // Lets build our docker image
        dir ('sample-bank-app-service') {
            def app = docker.build("sample-bankapp-service:${BUILD_NUMBER}")
        }
    }
    
    stage('CleanStaging') {
        // The cleanup script makes sure no previous docker staging containers run
        dir ('sample-bank-app-service') {
            sh "./cleanup.sh SampleOnlineBankStaging"
        }
    }
    
    stage('DeployStaging') {
        // Lets deploy the previously build container
        def app = docker.image("sample-bankapp-service:${BUILD_NUMBER}")
        app.run("--name SampleOnlineBankStaging -p 3000:3000 " +
                "-e 'DT_CLUSTER_ID=SampleOnlineBankStaging' " + 
                "-e 'DT_TAGS=Environment=Staging Service=Sample-NodeJs-Service' " +
                "-e 'DT_CUSTOM_PROP=ENVIRONMENT=Staging JOB_NAME=${JOB_NAME} " + 
                    "BUILD_TAG=${BUILD_TAG} BUILD_NUMBER=${BUIlD_NUMBER}'")

        dir ('dynatrace-scripts') {
            // push a deployment event on the host with the tag [AWS]Environment:JenkinsTutorial
            sh './pushdeployment.sh HOST CONTEXTLESS Jenkins ' +
               '${BUILD_TAG} ${BUILD_NUMBER} ${JOB_NAME} ' + 
               'Jenkins ${JENKINS_URL} ${JOB_URL} ${BUILD_URL} ${GIT_COMMIT}'
            
            // now I push one on the actual service (it has the tags from our rules)
            sh './pushdeployment.sh SERVICE CONTEXTLESS DockerService SampleOnlineBankStaging ' + 
               '${BUILD_TAG} ${BUILD_NUMBER} ${JOB_NAME} ' + 
               'Jenkins ${JENKINS_URL} ${JOB_URL} ${BUILD_URL} ${GIT_COMMIT}'
        }    
    }
    
    stage('Testing') {
        // lets push an event to dynatrace that indicates that we START a load test
        dir ('dynatrace-scripts') {
            sh './pushevent.sh SERVICE CONTEXTLESS DockerService SampleOnlineBankStaging ' +
               '"STARTING Load Test" ${JOB_NAME} "Starting a Load Test as part of the Testing stage"' + 
               ' ${JENKINS_URL} ${JOB_URL} ${BUILD_URL} ${GIT_COMMIT}'
        }
        
        // lets run some test scripts
        dir ('sample-nodejs-service-tests') {
            // start load test and run for 120 seconds - simulating traffic for Staging enviornment on port 80
            sh "rm -f stagingloadtest.log stagingloadtestcontrol.txt"
            sh "./loadtest.sh 3000 stagingloadtest.log stagingloadtestcontrol.txt 1 Staging"
            
            archiveArtifacts artifacts: 'stagingloadtest.log', fingerprint: true
        }

        // lets push an event to dynatrace that indicates that we STOP a load test
        dir ('dynatrace-scripts') {
            sh './pushevent.sh SERVICE CONTEXTLESS DockerService SampleOnlineBankStaging '+
               '"STOPPING Load Test" ${JOB_NAME} "Stopping a Load Test as part of the Testing stage" '+
               '${JENKINS_URL} ${JOB_URL} ${BUILD_URL} ${GIT_COMMIT}'
        }
    }
    
    stage('ValidateStaging') {
        // lets see if Dynatrace AI found problems -> if so - we can stop the pipeline!
        dir ('dynatrace-scripts') {
            DYNATRACE_PROBLEM_COUNT = sh 'python3 checkforproblems.py ${DT_URL} ${DT_TOKEN}'
            echo "Dynatrace Problems Found: ${DYNATRACE_PROBLEM_COUNT}"
        }
        
        // now lets generate a report using our CLI and lets generate some direct links back to dynatrace
        dir ('dynatrace-cli') {
            sh 'python3 dtcli.py dqlr srv tags/CONTEXTLESS:DockerService=SampleOnlineBankStaging '+
                        'service.responsetime[avg%hour],service.responsetime[p90%hour]'
            sh 'mv dqlreport.html dqlstagingreport.html'
            archiveArtifacts artifacts: 'dqlstagingreport.html', fingerprint: true
            
            // get the link to the service's dashboard and make it an artifact
            sh 'python3 dtcli.py link srv tags/CONTEXTLESS:DockerService=SampleOnlineBankStaging '+
                        'overview 60:0 ${DT_URL} ${DT_TOKEN} > dtstagelinks.txt'
            archiveArtifacts artifacts: 'dtstagelinks.txt', fingerprint: true
        }
    }
    
    stage('DeployProduction') {
        // first we clean production
        dir ('sample-bank-app-service') {
            sh "./cleanup.sh SampleOnlineBankStaging"
        }

        // now we deploy the new container
        def app = docker.image("sample-bankapp-service:${BUILD_NUMBER}")
        app.run("--name SampleOnlineBankStaging -p 3000:3000 "+
                "-e 'DT_CLUSTER_ID=SampleOnlineBankProduction' "+
                "-e 'DT_TAGS=Environment=Production Service=Sample-NodeJs-Service' "+
                "-e 'DT_CUSTOM_PROP=ENVIRONMENT=Production JOB_NAME=${JOB_NAME} "+
                    "BUILD_TAG=${BUILD_TAG} BUILD_NUMBER=${BUIlD_NUMBER}'")

        dir ('dynatrace-scripts') {
            // push a deployment event on the host with the tag [AWS]Environment:JenkinsTutorial
            sh './pushdeployment.sh HOST AWS Environment JenkinsTutorial '+
               '${BUILD_TAG} ${BUILD_NUMBER} ${JOB_NAME} Jenkins '+
               '${JENKINS_URL} ${JOB_URL} ${BUILD_URL} ${GIT_COMMIT}'
            
            // now I push one on the actual service (it has the tags from our rules)
            sh './pushdeployment.sh SERVICE CONTEXTLESS DockerService SampleNodeJsProduction '+
               '${BUILD_TAG} ${BUILD_NUMBER} ${JOB_NAME} Jenkins '+
               '${JENKINS_URL} ${JOB_URL} ${BUILD_URL} ${GIT_COMMIT}'
        }    
    }    
    
    stage('WarmUpProduction') {
        // lets push an event to dynatrace that indicates that we START a load test
        dir ('dynatrace-scripts') {
            sh './pushevent.sh SERVICE Environment Service Sample-NodeJs-Service '+
               '"STARTING Load Test" ${JOB_NAME} "Starting a Load Test to warm up new prod deployment" '+
               '${JENKINS_URL} ${JOB_URL} ${BUILD_URL} ${GIT_COMMIT}'
        }
        
        // lets run some test scripts
        dir ('sample-nodejs-service-tests') {
            // start load test and run for 120 seconds - simulating traffic for Production enviornment on port 90
            sh "rm -f productionloadtest.log productionloadtestcontrol.txt"
            sh "./loadtest.sh 90 productionloadtest.log productionloadtestcontrol.txt 60 Production"
            
            archiveArtifacts artifacts: 'productionloadtest.log', fingerprint: true
        }

        // lets push an event to dynatrace that indicates that we STOP a load test
        dir ('dynatrace-scripts') {
            sh './pushevent.sh SERVICE Environment Service Sample-NodeJs-Service '+
               '"STOPPING Load Test" ${JOB_NAME} "Stopping a Load Test as part of the Production warm up phase" '+
               '${JENKINS_URL} ${JOB_URL} ${BUILD_URL} ${GIT_COMMIT}'
        }
    }
    
    stage('ValidateProduction') {
        dir ('dynatrace-scripts') {
            DYNATRACE_PROBLEM_COUNT = sh (script: './checkforproblems.sh', returnStatus : true)
            echo "Dynatrace Problems Found: ${DYNATRACE_PROBLEM_COUNT}"
        }
        
        // now lets generate a report using our CLI and lets generate some direct links back to dynatrace
        dir ('dynatrace-cli') {
            sh 'python3 dtcli.py dqlr srv tags/CONTEXTLESS:DockerService=SampleNodeJsProduction '+
               'service.responsetime[avg%hour],service.responsetime[p90%hour] ${DT_URL} ${DT_TOKEN}'
            sh 'mv dqlreport.html dqlproductionreport.html'
            archiveArtifacts artifacts: 'dqlproductionreport.html', fingerprint: true

            // sh 'python3 dtcli.py link srv tags/CONTEXTLESS:DockerService=SampleNodeJsProduction ' +
            //    ' overview 60:0 ${DT_URL} ${DT_TOKEN} > dtprodlinks.txt'
            // archiveArtifacts artifacts: 'dtprodlinks.txt', fingerprint: true
        }
    }    
}