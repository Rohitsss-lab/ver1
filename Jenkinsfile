pipeline {
    agent any
    tools {
        nodejs 'NodeJS-20'
    }
    parameters {
        string(name: 'BUMP_TYPE',           defaultValue: 'patch', description: 'Version bump type')
        string(name: 'DEPLOY_TAG',          defaultValue: '',      description: 'Tag to deploy — leave blank for normal build')
        string(name: 'TRIGGERED_BY_DEPLOY', defaultValue: 'false', description: 'Set true when called from vertot deploy')
    }
    environment {
        GIT_REPO_URL = 'https://github.com/Rohitsss-lab/ver1.git'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout') {
            steps {
                script {
                    if (params.DEPLOY_TAG?.trim()) {
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: "refs/tags/v${params.DEPLOY_TAG}"]],
                            userRemoteConfigs: [[
                                url: "${env.GIT_REPO_URL}",
                                credentialsId: 'github-token'
                            ]]
                        ])
                        echo "Checked out ver1 at tag v${params.DEPLOY_TAG}"
                    } else {
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: '*/main']],
                            userRemoteConfigs: [[
                                url: "${env.GIT_REPO_URL}",
                                credentialsId: 'github-token'
                            ]]
                        ])
                    }
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                bat 'npm install'
            }
        }
        stage('Run Tests') {
            when {
                expression { return params.DEPLOY_TAG == null || params.DEPLOY_TAG.trim() == '' }
            }
            steps {
                bat 'npm test'
            }
        }
        stage('Bump Version') {
            when {
                expression { return params.DEPLOY_TAG == null || params.DEPLOY_TAG.trim() == '' }
            }
            steps {
                script {
                    withEnv(["BUMP_TYPE=${params.BUMP_TYPE}"]) {
                        def output = bat(
                            script: '"C:\\Program Files\\Python313\\python.exe" bump_version.py',
                            returnStdout: true
                        ).trim()
                        echo "Python output: ${output}"
                        def lines = output.split('\n')
                        def versionLine = lines.find { it.trim().startsWith('NEW_VERSION=') }
                        if (versionLine) {
                            env.NEW_VERSION = versionLine.trim().replace('NEW_VERSION=', '').trim()
                        } else {
                            env.NEW_VERSION = readFile('NEW_VERSION.txt').trim()
                        }
                    }
                    echo "==========================================="
                    echo "BUMPED VERSION = ${env.NEW_VERSION}"
                    echo "==========================================="
                    if (!env.NEW_VERSION || env.NEW_VERSION == '') {
                        error "Version bump failed — NEW_VERSION is empty"
                    }
                }
            }
        }
        stage('Commit and Push') {
            when {
                expression { return params.DEPLOY_TAG == null || params.DEPLOY_TAG.trim() == '' }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-token',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    bat '''
                        git config user.email "jenkins@ci.com"
                        git config user.name "Jenkins"
                        git checkout -b release/v%NEW_VERSION%
                        git add versions.json package.json
                        git commit -m "chore: bump version to v%NEW_VERSION%"
                        git remote set-url origin https://%GIT_USER%:%GIT_TOKEN%@github.com/Rohitsss-lab/ver1.git
                        git push origin release/v%NEW_VERSION%
                        git checkout main
                        git merge release/v%NEW_VERSION%
                        git push origin main
                        git tag v%NEW_VERSION% || echo "Tag already exists skipping"
                        git push origin v%NEW_VERSION% || echo "Tag already pushed skipping"
                    '''
                }
            }
        }
        stage('Deploy') {
            when {
                expression { return params.DEPLOY_TAG != null && params.DEPLOY_TAG.trim() != '' }
            }
            steps {
                echo "==========================================="
                echo "DEPLOYING ver1 at tag ${params.DEPLOY_TAG}"
                echo "==========================================="
                bat 'npm install'
                echo "ver1 v${params.DEPLOY_TAG} deployed on port 3001"
            }
        }
        stage('Notify vertot') {
            when {
                allOf {
                    expression { return params.DEPLOY_TAG == null || params.DEPLOY_TAG.trim() == '' }
                    expression { return params.TRIGGERED_BY_DEPLOY == 'false' }
                }
            }
            steps {
                script {
                    echo "==========================================="
                    echo "NEW_VERSION = ${env.NEW_VERSION}"
                    echo "Sending ver1 version ${env.NEW_VERSION} to vertot"
                    echo "==========================================="
                    if (!env.NEW_VERSION || env.NEW_VERSION == '' || env.NEW_VERSION == '1.0.0') {
                        error "Refusing to send wrong version '${env.NEW_VERSION}' to vertot"
                    }
                    build job: 'vertot',
                          wait: true,
                          parameters: [
                              string(name: 'REPO_NAME',      value: 'ver1'),
                              string(name: 'REPO_VERSION',   value: env.NEW_VERSION),
                              string(name: 'BUMP_TYPE',      value: 'patch'),
                              string(name: 'DEPLOY_VERSION', value: '')
                          ]
                }
            }
        }
    }
    post {
        success { echo "ver1 pipeline completed successfully" }
        failure { echo "ver1 pipeline FAILED" }
    }
}
