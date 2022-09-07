#!groovy
@Library('jenkins-pipeline-shared') _

pipeline {

    agent none

    environment {
        linuxDockerRegistry = "quay.io/ayfie"
        windowsDockerRegistry = "ayfiehub"
        NUGET_API_KEY = credentials('ayfie_jenkins_nexus_nugetapikey')
        NEXUS_CREDENTIALS = credentials('nexus')
        NUGET_CERT_REVOCATION_MODE = 'offline'
        BUILDCONFIG = 'Release'
    }

    options {
        gitLabConnection('git.ayfie.cloud')
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        disableConcurrentBuilds()
        quietPeriod(300)
    }

    triggers {
        gitlab(triggerOnPush: true, triggerOnMergeRequest: true, branchFilterType: 'All')

        // Trigger daily at 02:00, but only for master branch
        cron(env.BRANCH_NAME == 'master' ? 'H 2 * * *' : '')
    }

    stages {
        stage('Parallel Stage') {

            parallel {

                stage("Windows Node") {

                    agent { label "windows" }

                    stages {
                        stage("Windows: Checkout") {
                            steps {
                                checkout scm
                                bat 'git status'
                                bat 'git branch -vva'
                            }
                        }

                        stage("Windows: Clean packages folder") {
                            steps {
                                dir('packages') {
                                    deleteDir()
                                }
                            }
                        }

                        stage("Windows: Build") {
                            steps {
                                updateGitlabCommitStatus name: 'build', state: 'running'
                                echo "Executing PowerShell script"
                                echo "NUGET_CERT_REVOCATION_MODE=${NUGET_CERT_REVOCATION_MODE}"
                                // Get how much disk space is available on all local drives
                                powershell("Get-WmiObject -Class Win32_logicaldisk")
                                powershell(".\\build.ps1 BuildDockerImageForWindows PublishConnectorEntrypointInDocker --configuration ${BUILDCONFIG}")
                                echo "Executed PowerShell script."
                            }
                        }

                        stage("Windows: Test") {
                            steps {
                              echo "Executing PowerShell script"
                              powershell(".\\build.ps1 UnitTestsInDocker --configuration ${BUILDCONFIG}")
                              echo "Executed PowerShell script."
                            }
                        }

                        stage("Windows: Build Connector Runtime Image") {
                            steps {
                                echo "NUGET_CERT_REVOCATION_MODE=${NUGET_CERT_REVOCATION_MODE}"
                                powershell(".\\build.ps1 BuildDockerRuntimeImageForWindows --configuration ${BUILDCONFIG}")
                                echo "Executed PowerShell script."
                            }
                        }

                        stage("Windows: IntegrationTest") {
                            steps {
                                echo "Executing PowerShell script"
                                powershell(".\\build.ps1 IntegrationTestsInDocker --configuration ${BUILDCONFIG}")
                                echo "Executed PowerShell script."
                            }
                        }

                        stage("Windows: Docker Publish") {
                            steps {
                                powershell(".\\build.ps1 DockerPushToDockerHub")
                            }
                        }

                        stage("Windows: Build and Publish Installation Bundle") {
                            steps {
                                powershell(".\\build.ps1 BuildInstallationBundle PublishInstallationBundle")
                            }
                        }

                        stage("Windows: Clean Images") {
                            steps {
                              echo "Executing PowerShell script"
                              powershell(".\\build.ps1 CleanImages --configuration ${BUILDCONFIG}")
                              echo "Executed PowerShell script."
                            }
                        }

                    }
                    post {
                        always {
                            echo "Cleanup of Windows Node step"

                            dir("tmp/context/testresults") {
                                mstest testResultsFile:"**/*.trx", keepLongStdio: true
                                archiveArtifacts artifacts: '**/*.log, Sequence.xml', fingerprint: true, allowEmptyArchive: true
                            }

                        }
                    }
                }
            }
        }
    }
    post {
        always {
            // Send notification
            sendNotification currentBuild.result
        }
        failure {
            updateGitlabCommitStatus name: 'build', state: 'failed'

            script{
                if (env.BRANCH_NAME == 'master' || env.BRANCH_NAME == 'develop') {
                    slackSend channel: '#jenkins-saga-pipeline',
                        color: 'danger',
                        teamDomain: 'ayfie',
                        tokenCredentialId: '4ef80452-7d7c-494c-a5fe-27ae59fcec80',
                        username: 'Jenkins Notifier',
                        message: "*${currentBuild.currentResult}:* @here Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} failed! \n More info at: ${env.BUILD_URL}"

                    emailext body: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} failed! \n More info at: ${env.BUILD_URL}",
                         recipientProviders: [[$class: 'DevelopersRecipientProvider'],
                                            [$class: 'RequesterRecipientProvider'],
                                            [$class: 'CulpritsRecipientProvider'],
                                            [$class: 'UpstreamComitterRecipientProvider']],
                         subject: "Build ${env.JOB_NAME} failed!"
                }
            }
        }
        success {
            updateGitlabCommitStatus name: 'build', state: 'success'
        }
        aborted {
            updateGitlabCommitStatus name: 'build', state: 'canceled'
        }
    }
}
