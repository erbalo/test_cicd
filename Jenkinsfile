def serviceName = 'test_cicd_front'

def region = 'us-east-1'
def buildBucket = 'test-jenkins-manager'
def devBucket = 'test-react-cicd.casai.com'

def version = null
def artifactFilename = null

pipeline {
    agent {
        docker {
            image 'node:12-alpine'
            args '-p 3000:3000'
        }
    }

    environment {
        CI = 'true'
        HOME = '.'
        npm_config_cache = 'npm-cache'
    }

    stages {
        stage('Pull Source') {
            steps {
                checkout scm
            }
        }

        stage('Install Packages') {
            steps {
                sh 'npm install'
            }
        }

        stage('Test') {
            steps {
                sh 'npm run test'
            }
        }

        stage('Create Version') {
            steps {
                script {
                    props = readJSON file: 'package.json'
                    version = props.version
                    echo "Version ${version}"
                }
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Deploy To Dev') {
            steps {
                withAWS(region: region, credentials:'id_aws_erbalo') {
                    s3Upload(bucket: devBucket, workingDir: 'build', path: '', includePathPattern:'**/*', cacheControl:'no-cache,no-store')
                }
            }
        }

        stage('Build Artifact') {
            steps {
                script {
                    artifactFilename = "${version}.tar.gz"
                    sh "tar czf ${artifactFilename} build/"

                    withAWS(region: region, credentials:'id_aws_erbalo') {
                        s3Upload(bucket: buildBucket, file: artifactFilename, path: "${serviceName}/")
                    }

                    sh "rm ${artifactFilename}"
                }

                slackSend(
                    channel: '#test-tech-integrations',
                    color: 'good',
                    message: "Build - ${currentBuild.number}: [${serviceName}] Generating release ${version}"
                )
            }
        }
    }
}
