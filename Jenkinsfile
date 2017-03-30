#!groovy
// vim: ft=groovy

// Build environment
ENVIRONMENT = env.JOB_NAME.tokenize('-')[-1]
CHECKOUT_REFS = null

pipeline {
    agent none
    options {
        skipDefaultCheckout()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    parameters {
        string(
            name: 'CHECKOUT_REFS',
            defaultValue: '',
            description: ''
        )
    }

    stages {
        stage ('Prepare') {
            agent { label 'master' }
            steps {
                echo 'hello'
            }
        }

        stage ('Checkout') {
            agent any
            steps {
                script {
                    CHECKOUT_REFS = params.CHECKOUT_REFS ?: scm.branches[0].name
                }

                checkout([
                    $class: 'GitSCM',
                    branches: [[name: CHECKOUT_REFS]],
                    extensions: [
                        [$class: 'CloneOption', depth: 0, noTags: false, reference: '', shallow: false, timeout: 5],
                        [$class: 'PruneStaleBranch'],
                        [$class: 'AuthorInChangelog']
                    ],
                    userRemoteConfigs: [
                        [
                            name: 'origin',
                            url: scm.userRemoteConfigs[0].url,
                            refspec: '+refs/heads/*:refs/remotes/origin/* +refs/pull/*/head:refs/remotes/origin/PR/*',
                            credentialsId: scm.userRemoteConfigs[0].credentialsId,
                        ]
                    ]
                ])
            }
        }

        stage('Build') {
            agent { docker 'openjdk:8u121-jdk' }
            steps {
                sh 'date > a.out'
                stash name: 'app', includes: "a.out"
            }
        }


        stage('Deploy') {
            agent any
            steps {
                lock("deploy:testing:${ENVIRONMENT}") {
                    unstash name: 'app'
                    sh 'true'
                } /*lock*/
            } /* steps */
        } /* stage */
    }


    post {
        success {
            node('master') {
               unstash name: 'app'
               archiveArtifacts \
                   artifacts: "a.out",
                   onlyIfSuccessful: true
            }

            echo "number: ${currentBuild.number}"
            echo "displayName: ${currentBuild.displayName}"
            echo "description: ${currentBuild.description}"
            echo "id: ${currentBuild.id}"
            //slackSend \
            //    message: "배포 성공: ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)",
            //        color: "good"
        }

        failure {
            echo 'fail'
            //slackSend \
            //    message: "빌드 실패: ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)",
            //        color: "danger"
        }
    }

}
