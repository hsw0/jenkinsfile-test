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
                sh './gradlew --version'

                sh([
                        "./gradlew",
                        "--gradle-user-home=.gradle/home-local",  // per-project gradle home
                        "--no-daemon",
                        "--stacktrace",
                        "clean demo:assemble"
                ].join(' '))

                stash name: 'app', includes: "demo/build/libs/*.jar"
            }
        }


        stage('Tests') {
            agent { docker 'openjdk:8u121-jdk' }
            when { expression { ENVIRONMENT != 'live' } }
            steps {
                sh([
                        "./gradlew",
                        "--gradle-user-home=.gradle/home-local",  // per-project gradle home
                        "--no-daemon",
                        "--stacktrace",
                        "demo:test"
                ].join(' '))

                junit allowEmptyResults: true, testResults: 'demo/build/test-results/test/*.xml'
            }
        }


        stage('Deploy') {
            agent any
            when { expression { ENVIRONMENT != 'test' } }
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
                    artifacts: "demo/build/libs/*.jar",
                    onlyIfSuccessful: true

                // Add deploy tag
                sh 'git tag -a -m "${JOB_NAME} b${BUILD_NUMBER}" "deploy/${BUILD_TAG}"'

                sshagent([scm.userRemoteConfigs[0].credentialsId]) {
                    sh 'git push origin "deploy/${BUILD_TAG}"'
                }
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
