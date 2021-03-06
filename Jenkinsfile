#!groovy

import groovy.json.JsonSlurper

pipeline {
    agent any
    stages {
        stage('Authorize PROD') {
            when { branch 'master' }
            steps {
                script {
                    rc = sh returnStatus: true, script: "sfdx force:auth:jwt:grant --clientid ${env.DEV_HUB_CONSUMER_KEY} --username ${env.DEV_HUB_USERNAME} --jwtkeyfile build/server.key --setalias PROD"
                    if (rc != 0) { error 'hub org authorization failed' }
                }
            }
        }
        stage('Deploy to PROD') {
            when { branch 'master' }
            stages {
                stage('Validate package') {
                    steps {
                        script {
                            // convert to metadata api
                            rc = sh returnStatus: true, script: "sfdx force:source:convert --rootdir force-app/ --outputdir src/"
                            if (rc != 0) { error 'metadata convert failed' }
                            rmsg = sh returnStdout: true, script: "sfdx force:mdapi:deploy --checkonly --deploydir src/ --targetusername PROD --testlevel RunLocalTests --wait 10 --json"
                            def robj = new JsonSlurper().parseText(rmsg)
                            if (robj.status != 0) { error 'prod deploy failed: ' + robj.message }
                            else { env.VALIDATION_ID = robj.result.id }
                            // assign permset
                            // rc = sh returnStatus: true, script: "sfdx force:user:permset:assign --targetusername ${SFDC_USERNAME} --permsetname DreamHouse"
                            // if (rc != 0) {
                            //     error 'permset:assign failed'
                            // }
                        }
                    }
                }
                stage('Quick deploy') {
                    input {
                        message 'Commit deploy?'
                        ok 'Yes'
                        parameters {
                            booleanParam(name: 'COMMIT', defaultValue: true, description: '')
                        }
                    }
                    steps {
                        script {
                            rc = sh returnStatus: true, script: "sfdx force:mdapi:deploy --targetusername PROD --validateddeployrequestid ${env.VALIDATION_ID}"
                            if (rc != 0) { error 'quick deploy failed' }
                        }
                    }
                }
            }
        }
        stage('Authorize STAGE') {
            when { branch 'stage' }
            steps {
                script {
                    rc = sh returnStatus: true, script: "sfdx force:auth:jwt:grant --clientid ${env.STAGE_CONSUMER_KEY} --username ${env.STAGE_USERNAME} --jwtkeyfile build/server.key --instanceurl https://test.salesforce.com --setalias STAGE"
                    if (rc != 0) { error 'hub org authorization failed' }
                }
            }
        }
        stage('Deploy to STAGE') {
            when { branch 'stage' }
            stages {
                stage('Validate STAGE package') {
                    steps {
                        script {
                            // convert to metadata api
                            rc = sh returnStatus: true, script: "sfdx force:source:convert --rootdir force-app/ --outputdir src/"
                            if (rc != 0) { error 'metadata convert failed' }
                            rc = sh returnStatus: true, script: "sfdx force:mdapi:deploy --deploydir src/ --targetusername STAGE --testlevel RunLocalTests --wait 10 --json"
                            if (rc != 0) { error 'stage deploy failed' }
                            // assign permset
                            // rc = sh returnStatus: true, script: "sfdx force:user:permset:assign --targetusername ${SFDC_USERNAME} --permsetname DreamHouse"
                            // if (rc != 0) {
                            //     error 'permset:assign failed'
                            // }
                        }
                    }
                }
            }
        }
        stage('Authorize DevHub') {
            when { branch 'feature*' }
            steps {
                script {
                    rc = sh returnStatus: true, script: "sfdx force:auth:jwt:grant --clientid ${env.DEV_HUB_CONSUMER_KEY} --username ${env.DEV_HUB_USERNAME} --jwtkeyfile build/server.key --setdefaultdevhubusername"
                    if (rc != 0) { error 'hub org authorization failed' }
                }
            }
        }
        stage('Build') {
            when { branch 'feature*' }
            steps {
                script {
                    rc = sh returnStatus: true, script: "sfdx force:org:create --definitionfile config/project-scratch-def.json --json --setdefaultusername --setalias ciorg --durationdays 1"
                    if (rc != 0) { error 'scratch org creation failed' }
                    rc = sh returnStatus: true, script: "sfdx force:lightning:test:install --packagetype jasmine --wait 5"
                    if (rc != 0) { error 'Lightning Testing Service install failed' }
                    rc = sh returnStatus: true, script: "sfdx force:source:push"
                    if (rc != 0) { error 'source push failed' }
                    // assign permset
                    // rc = sh returnStatus: true, script: "sfdx force:user:permset:assign --targetusername ${SFDC_USERNAME} --permsetname DreamHouse"
                    // if (rc != 0) {
                    //     error 'permset:assign failed'
                    // }
                }
            }
        }
        stage('Tests') {
            when { branch 'feature*' }
            failFast true
            parallel {
                stage('Run Apex tests') {
                    options {
                        timeout(time: 120, unit: 'SECONDS') 
                    }
                    steps {
                        script {
                            rc = sh returnStatus: true, script: "sfdx force:apex:test:run --testlevel RunLocalTests --codecoverage --outputdir tests/ --resultformat junit"
                            if (rc != 0) { error 'apex test run failed' }
                        }
                    }
                }
                stage('Run Aura tests') {
                    options {
                        timeout(time: 120, unit: 'SECONDS')
                    }
                    steps {
                        script {
                            rc = sh returnStatus: true, script: "sfdx force:lightning:test:run -a myTestSuite.app"
                            if (rc != 0) { error 'aura test run failed' }
                        }
                    }
                }
                // stage('Run LWC tests') {
                //     options {
                //         timeout(time: 120, unit: 'SECONDS')
                //     }
                //     steps {
                //         script {
                //             rc = sh returnStatus: true, script: "npm run jest"
                //             if (rc != 0) { error 'lwc test run failed' }
                //         }
                //     }
                // }
            }
        }
        stage('Reports') {
            when { branch 'feature*' }
            steps {
                junit keepLongStdio: true, testResults: 'tests/*-junit.xml'
            }
        }
    }
    post {
        cleanup {
            echo 'sfdx ciorg delete'
            script {
                rc = sh returnStatus: true, script: "sfdx force:org:delete --targetusername ciorg --noprompt"
            }
        }
    }
}