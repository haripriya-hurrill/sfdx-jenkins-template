#!groovy

node {
    stage('checkout source') {
        checkout scm
    }

    stage('deploy') {
        if (env.BRANCH_NAME == 'master') { 
            echo 'prod build'
        }

        if (env.BRANCH_NAME == 'stage') {
            echo 'stage build'
        }

        if (env.BRANCH_NAME ==~ /feature\.*/) {
            echo 'sfdx ci build'
        }
    }
}