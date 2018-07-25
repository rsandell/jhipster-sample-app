pipeline {
    environment {
        REL_VERSION = "${TAG_NAME?.contains('release-') ? TAG_NAME.drop(TAG_NAME.lastIndexOf('-')+1) + '.' + BUILD_NUMBER : 'M.' + BUILD_NUMBER}"
    }
    agent none
    options {
        skipDefaultCheckout()
    }
    stages {
        stage('Checkout') {
            agent any
            steps {
                checkout scm
                stash(name: 'ws', includes: '**', excludes: '**/.git/**')
            }
        }
        stage('Build Backend') {
            agent {
                docker {
                    image 'maven:3-alpine'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                unstash 'ws'
                sh 'mvn -B -DskipTests=true clean compile package'
                stash name: 'war', includes: 'target/**/*'
            }
        }
        stage('Test Backend') {
            agent {
                docker {
                    image 'maven:3-alpine'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                unstash 'ws'
                unstash 'war'
                sh 'mvn -B test findbugs:findbugs'
            }
            post {
                success {
                    junit '**/surefire-reports/**/*.xml'
                    findbugs pattern: 'target/**/findbugsXml.xml', unstableNewAll: '0' //unstableTotalAll: '0'
                }
                unstable {
                    junit '**/surefire-reports/**/*.xml'
                    findbugs pattern: 'target/**/findbugsXml.xml', unstableNewAll: '0' //unstableTotalAll: '0'
                }
            }
        }
        stage('Test More') {
            parallel {
                stage('Frontend') {
                    when {
                        anyOf {
                            branch "master"
                            tag "release-*"
                            changeset "src/main/webapp/**/*"
                        }
                    }
                    agent {
                        dockerfile {
                            args "-v /tmp:/tmp"
                            dir "docker/gulp"
                        }
                    }
                    steps {
                        unstash 'ws'
                        unstash 'war'
                        sh '. target/scripts/frontEndTests.sh'
                    }
                }
                stage('Performance') {
                    when {
                        allOf {
                            anyOf {
                                branch "master"
                                tag "release-*"
                            }
                            not {
                                changeRequest()
                            }
                        }
                    }
                    agent {
                        docker {
                            image 'maven:3-alpine'
                            args '-v $HOME/.m2:/root/.m2'
                        }
                    }
                    steps {
                        unstash 'ws'
                        unstash 'war'
                        sh 'mvn -B gatling:execute'
                    }
                }
            }
        }
        stage('Deploy to Staging') {
            agent any
            environment {
                STAGING_AUTH = credentials('staging')
            }
            when {
                anyOf {
                    branch "master"
                    changeRequest target: "master"
                    not {
                        buildingTag()
                    }
                }
            }
            steps {
                unstash 'war'
                sh '. target/scripts/deploy.sh staging -v $REL_VERSION -u $STAGING_AUTH_USR -p $STAGING_AUTH_PSW'
            }
            //Post: Send notifications; hipchat, slack, send email etc.
        }
        stage('Archive') {
            agent any
            when {
                not {
                    anyOf {
                        branch "master"
                        buildingTag()
                    }
                }
            }
            steps {
                unstash 'war'
                archiveArtifacts artifacts: 'target/**/*.war', fingerprint: true, allowEmptyArchive: true
            }
        }
        stage('Deploy to Production') {
            agent { label 'linux' }
            environment {
                PROD_AUTH = credentials('production')
            }
            when {
                tag "release-*"
            }
            options {
                timeout(15)
            }
            input {
                message 'Deploy to production?'
                ok 'Fire zee missiles!'
            }
            steps {
                unstash 'war'
                sh '. target/scripts/deploy.sh production -v $REL_VERSION -u $PROD_AUTH_USR -p $PROD_AUTH_PSW'
            }
        }
    }
    //Post: notifications; hipchat, slack, send email etc.
}
