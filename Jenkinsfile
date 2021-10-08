def serviceName = 'test_cicd_front'

def region = 'us-east-1'
def buildBucket = 'test-jenkins-manager'

def version = null
def artifactFilename = null
def previewURL = null


pipeline {
    agent {
        docker {
            image 'node:12-alpine'
            args '-p 3000 -u root --privileged'
        }
    }

    environment {
        CI = 'true'
        HOME = '.'
        npm_config_cache = 'npm-cache'
        FIREBASE_DEPLOY_TOKEN = credentials('firebase-deploy-token')
        GIT_COMMIT_SHORT = env.GIT_COMMIT.take(7)
    }

    stages {
        stage('Pull Source') {
            steps {
                checkout scm
            }
        }

        stage('Install Packages') {
            steps {
                githubNotify context: 'CI', description: 'Installing packages...', status: 'PENDING'
                sh 'npm install'
                sh 'npm install -g firebase-tools'
            }
        }

        stage('Test') {
            steps {
                githubNotify context: 'CI', description: 'Running testing...',  status: 'PENDING'
                sh 'npm run test'
            }
        }

        stage('Create Version') {
            steps {
                script {
                    props = readJSON file: 'package.json'
                    version = props.version
                    echo "Version ${version}"
                    echo "Commit ${GIT_COMMIT_SHORT}"
                }
            }
        }

        stage('Build') {
            steps {
                githubNotify context: 'CI', description: 'Building project...',  status: 'PENDING'
                sh 'npm run build'
            }
        }

        stage('Building Preview') {
            steps {
                githubNotify context: 'CD', description: 'Building preview...',  status: 'PENDING'
                script {
                    previewProps = sh(
                        script: 'firebase hosting:channel:deploy preview-$GIT_COMMIT_SHORT --expires 3d --token $FIREBASE_DEPLOY_TOKEN',
                        returnStdout: true
                    )

                    try {
                        lastLine = previewProps.tokenize('\n').last()

                        previewURL = sh(
                            script: "echo \"${lastLine}\" | grep -Eo '(https?)://[-A-Za-z0-9\\+&@#/%?=~_|!:,.;]*[-A-Za-z0-9\\+&@#/%=~_|]'",
                            returnStdout: true
                        )

                        echo "Preview URL ${previewURL}"
                    }catch (Exception e) {
                        echo "Error extracting URL ${e}"
                        githubNotify context: 'CI', description: 'Error al genear preview...',  status: 'FAILURE'
                    }
                }
            }
        }

        stage('Build Artifact') {
            steps {
                script {
                    artifactFilename = "${version}.tar.gz"
                    sh "tar czf ${artifactFilename} build/"

                    githubNotify context: 'CI', description: 'Building artifact...',  status: 'PENDING'
                    withAWS(region: region, credentials:'id_aws_erbalo') {
                        s3Upload(bucket: buildBucket, file: artifactFilename, path: "${serviceName}/")
                    }

                    sh "rm ${artifactFilename}"
                }

                githubNotify context: 'CD', description: "Preview OK! ${previewURL}",  status: 'SUCCESS'
                githubNotify context: 'CI', description: 'Everything is OK!',  status: 'SUCCESS'

                /*slackSend(
                    channel: '#test-tech-integrations',
                    color: 'good',
                    message: "Build - ${currentBuild.number}: [${serviceName}] Generating release ${version}"
                )*/
            }
        }
    }
}
