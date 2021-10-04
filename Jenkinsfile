def serviceName = 'test_cicd_front'
def version = null

def deployToS3(bkt) {
    sendFileToS3 {
        bucket = bkt
        file = 'dist'
        path = ''
        isRecursive = true
    }

    sendFileToS3 {
        bucket = bkt
        file = 'dist/index.html'
        path = ''
        cacheControl = 'no-cache,no-store'
    }
}

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
            //deleteDir()
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
                    version = sh (returnStdout: true, script: "npm version ${VERSION_TYPE}").trim()
                    echo "${version}"

                    slackSend(
                        channel: '#test-tech-integrations',
                        color: 'good',
                        message: "[${serviceName}] Generating release ${version}"
                    )

                    stash name: 'workspaceWithDependenciesAndVersionCommit', useDefaultExcludes: false
                }                
            }
        }
    }
}
