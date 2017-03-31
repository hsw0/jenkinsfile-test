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
            agent { docker {
                image 'openjdk:8u121-jdk'
                reuseNode true
            } }
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


        stage('Check') {
            agent { docker {
                image 'openjdk:8u121-jdk'
                reuseNode true
            } }
            when { expression { ENVIRONMENT != 'live' } }
            steps {
                parallel (
                    "Unit tests": {
                        sh([
                                "./gradlew",
                                "--gradle-user-home=.gradle/home-local",  // per-project gradle home
                                "--stacktrace",
                                "demo:test"
                        ].join(' '))

                        junit testResults: 'demo/build/test-results/test/*.xml', allowEmptyResults: true
                    },
                    "Checkstyle": {
                        sh([
                                "./gradlew",
                                "--gradle-user-home=.gradle/home-local",  // per-project gradle home
                                "--stacktrace",
                                "demo:checkstyleMain demo:checkstyleTest"
                        ].join(' '))

                        step([$class: 'CheckStylePublisher', pattern: 'demo/build/reports/checkstyle/*.xml'])
                    },
                    "Findbugs": {
                        sh([
                                "./gradlew",
                                "--gradle-user-home=.gradle/home-local",  // per-project gradle home
                                "--stacktrace",
                                "demo:findbugsMain demo:findbugsTest"
                        ].join(' '))

                        step([$class: 'FindBugsPublisher', pattern: 'demo/build/reports/findbugs/*.xml', unHealthy: ''])
                    },
                    "PMD": {
                        sh([
                                "./gradlew",
                                "--gradle-user-home=.gradle/home-local",  // per-project gradle home
                                "--stacktrace",
                                "demo:pmdMain demo:pmdTest"
                        ].join(' '))

                        step([$class: 'PmdPublisher', pattern: 'demo/build/reports/pmd/*.xml', unHealthy: ''])
                    },
                )
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


        stage('Tag') {
            agent any
            when { expression { ENVIRONMENT == 'live' } }
            steps {
                // Add deploy tag
                sh 'git tag -a -m "${JOB_NAME} b${BUILD_NUMBER}" "deploy/${BUILD_TAG}"'

                sshagent([scm.userRemoteConfigs[0].credentialsId]) {
                    sh 'git push origin "deploy/${BUILD_TAG}"'
                }

                // TODO: newrelic

                /*
                echo "number: ${currentBuild.number}"
                echo "displayName: ${currentBuild.displayName}"
                echo "description: ${currentBuild.description}"
                echo "id: ${currentBuild.id}"
                */
            }
        }
    }


    post {
        success {
            step([$class: 'AnalysisPublisher'])

            node('master') {
                unstash name: 'app'
                archiveArtifacts \
                    artifacts: "demo/build/libs/*.jar",
                    onlyIfSuccessful: true
            }

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
