pipeline {
    agent any
    environment {
// I am passing env variable here
        REPO_URL = 'https://github.com/sidjena01/npmbuild.git'
    }
    stages {
        stage('Build') {
            steps {
                echo 'Pull code and build'
                git credentialsId: 'github', url: "${REPO_URL}"
                // bat 'npm install'
                   sh 'npm install'

                // bat 'npm run build'
                   sh 'npm run build'
            }
        }
        stage('Audit Dependencies'){
            steps{
                dependencyCheckAnalyzer datadir: "${env.DEPENDENCY_DB_PATH}", hintsFile: '', includeCsvReports: false, includeHtmlReports: true, includeJsonReports: false, includeVulnReports: false, isAutoupdateDisabled: false, outdir: '', scanpath: 'node_modules', skipOnScmChange: false, skipOnUpstreamChange: false, suppressionFile: '', zipExtensions: ''
                dependencyCheckPublisher canComputeNew: false, defaultEncoding: '', healthy: '', pattern: '', unHealthy: ''
                archiveArtifacts allowEmptyArchive: true, artifacts: '**/dependency-check-report.*', onlyIfSuccessful: true
            }
        }
        stage('Sonar Analysis'){
            input {
                message "Confirm no new vulnerabilities have been discovered."
                ok "Confirm"
            }
            steps{
             // sh '''sonarproperties'''
                echo 'Vulnerability check passed'
            }
        }
        stage('Test') {
            steps {
                echo 'Run tests and publish results'
                bat 'npm run test'
            }
        }
        stage('Increment Version') {
            steps {
                bat 'npm version patch -m "jenkins-release: Deployed package version %%s" --force'
                bat "git push \"${REPO_URL}\" master --tags"
            }
        }
        stage('Deploy to Development') {
            environment {
                VERSION = bat(returnStdout: true, script: '@npm run get-version --silent').trim()
            }
            steps {
                echo 'Deploy package to Dev'
                bat "serverless deploy --alias DEV --region ${env.DEPLOY_REGION} --version ${VERSION}"
            }
        }
        stage('Deploy to QA') {
            environment {
                VERSION = bat(returnStdout: true, script: '@npm run get-version --silent').trim()
            }
            input {
                message "Promote package to QA?"
                ok "Promote"
                submitter "${env.QA_DEPLOY_SUBMITTERS}"
            }
            steps {
                echo 'Modify vars and promote package to QA'
                bat "serverless deploy --alias QA --region ${env.DEPLOY_REGION} --version ${VERSION}"
            }
        }
        stage('Deploy to UAT') {
            environment {
                VERSION = bat(returnStdout: true, script: '@npm run get-version --silent').trim()
            }
            input {
                message "Promote package to UAT?"
                ok "Promote"
                submitter "${env.UAT_DEPLOY_SUBMITTERS}"
            }
            steps {
                echo 'Modify vars and promote package to UAT'
                bat "serverless deploy --alias UAT --region ${env.DEPLOY_REGION} --version ${VERSION}"
            }
        }
        stage('Release to Production') {
            input {
                message "Release package?"
                ok "Release"
                submitter "${env.LIVE_DEPLOY_SUBMITTERS}"
            }
            environment {
                TAG = bat(returnStdout: true, script: '@npm run get-version --silent').trim()
            }
            steps {
                echo 'Modify vars and promote release package'
                bat "serverless deploy --alias LIVE --region ${env.LIVE_DEPLOY_REGION} --deployBucket ${env.LIVE_DEPLOY_BUCKET} --version ${TAG}"

                script {

                    try{
                        bat "git tag -a ${TAG}_Released ${GIT_COMMIT} -m \"jenkins-release: Jenkins Git plugin tagging with ${TAG}_Released\""
                        bat "git push \"${REPO_URL}\" master --tags"
                    }
                    catch(Exception e){
                        echo "Tagging commit failed. Tag probably already exists."
                    }
                }
            }
        }
    }
    post {
        always {
            publishHTML target: [
            allowMissing         : true,
            alwaysLinkToLastBuild: false,
            keepAll              : true,
            reportDir            : 'coverage',
            reportFiles          : 'index.html',
            reportName           : 'Test Report'
            ]
        }
        failure {
            slackSend color: 'danger', message: "Build ${currentBuild.currentResult} ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)"
        }
    }
}
