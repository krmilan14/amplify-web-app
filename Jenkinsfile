pipeline {
    agent {label 'Mac-agent-15.2'}

    environment {
        AWS_REGION = 'ap-southeast-2'
        AWS_PROFILE = 'milan'
        APP_ID = 'dqob02f8yg24r'
        MAIN_BRANCH = 'main'
        BACKUP_BRANCH = 'main-minus-one'
    }

    stages {
        stage('Trigger Main Branch Deployment') {
            steps {
                script {
                    echo "Triggering ${MAIN_BRANCH} branch deployment..."

                    def startJobOutput = sh(
                        script: """
                        aws amplify start-job \
                          --app-id $APP_ID \
                          --branch-name $MAIN_BRANCH \
                          --job-type RELEASE \
                          --region $AWS_REGION \
                          --profile $AWS_PROFILE \
                          --output json
                        """,
                        returnStdout: true
                    ).trim()

                    def json = readJSON text: startJobOutput
                    env.MAIN_JOB_ID = json.jobSummary.jobId
                    echo "Started deployment job ID: ${env.MAIN_JOB_ID}"
                }
            }
        }

        stage('Monitor Main Branch Deployment Status') {
            steps {
                script {
                    echo "Monitoring main branch deployment status..."

                    def maxAttempts = 20
                    def intervalSeconds = 30
                    def attempt = 0
                    def jobStatus = "PENDING"

                    while (attempt < maxAttempts) {
                        sleep time: intervalSeconds, unit: 'SECONDS'
                        def statusJson = sh(
                            script: """
                            aws amplify get-job \
                              --app-id $APP_ID \
                              --branch-name $MAIN_BRANCH \
                              --job-id ${env.MAIN_JOB_ID} \
                              --region $AWS_REGION \
                              --profile $AWS_PROFILE \
                              --output json
                            """,
                            returnStdout: true
                        ).trim()

                        def jobDetail = readJSON text: statusJson
                        jobStatus = jobDetail.job.summary.status
                        echo "Main branch job status: ${jobStatus}"

                        if (jobStatus in ['SUCCEED', 'FAILED', 'CANCELLED']) {
                            break
                        }

                        attempt++
                    }

                    if (jobStatus != 'SUCCEED') {
                        echo "Main branch deployment failed or timed out."
                    } else {
                        echo "Main branch deployment succeeded ✅"
                    }

                    env.MAIN_BRANCH_STATUS = jobStatus
                }
            }
        }

        stage('Trigger Backup Branch Deployment after Main Bracnh Failed') {
            when {
                expression { return env.MAIN_BRANCH_STATUS != 'SUCCEED' }
            }
            steps {
                script {
                    echo "Triggering backup branch deployment..."

                    def backupJobOutput = sh(
                        script: """
                        aws amplify start-job \
                          --app-id $APP_ID \
                          --branch-name $BACKUP_BRANCH \
                          --job-type RELEASE \
                          --region $AWS_REGION \
                          --profile $AWS_PROFILE \
                          --output json
                        """,
                        returnStdout: true
                    ).trim()

                    def backupJson = readJSON text: backupJobOutput
                    env.BACKUP_JOB_ID = backupJson.jobSummary.jobId
                    echo "Started backup deployment job ID: ${env.BACKUP_JOB_ID}"
                }
            }
        }

        stage('Monitor Backup Branch Deployment Status') {
            when {
                expression { return env.MAIN_BRANCH_STATUS != 'SUCCEED' }
            }
            steps {
                script {
                    echo "Monitoring backup branch deployment status..."

                    def maxAttempts = 20
                    def intervalSeconds = 30
                    def attempt = 0
                    def backupJobStatus = "PENDING"

                    while (attempt < maxAttempts) {
                        sleep time: intervalSeconds, unit: 'SECONDS'
                        def backupStatusJson = sh(
                            script: """
                            aws amplify get-job \
                              --app-id $APP_ID \
                              --branch-name $BACKUP_BRANCH \
                              --job-id ${env.BACKUP_JOB_ID} \
                              --region $AWS_REGION \
                              --profile $AWS_PROFILE \
                              --output json
                            """,
                            returnStdout: true
                        ).trim()

                        def backupJobDetail = readJSON text: backupStatusJson
                        backupJobStatus = backupJobDetail.job.summary.status
                        echo "Backup branch job status: ${backupJobStatus}"

                        if (backupJobStatus in ['SUCCEED', 'FAILED', 'CANCELLED']) {
                            break
                        }

                        attempt++
                    }

                    if (backupJobStatus == 'SUCCEED') {
                        echo "Backup branch deployment succeeded ✅"
                    } else {
                        error("Backup branch deployment failed!")
                    }

                    env.BACKUP_BRANCH_STATUS = backupJobStatus
                }
            }
        }
    }

    post {
        success {
            echo "Deployment process completed successfully!"
        }
        failure {
            echo "Deployment process failed at some stage!"
        }
    }
}
 
