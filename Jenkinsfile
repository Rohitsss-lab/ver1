pipeline {
    agent any
    tools {
        nodejs 'NodeJS-20'
    }
    parameters {
        string(name: 'BUMP_TYPE', defaultValue: 'patch', description: 'Version bump type')
    }
    environment {
        REPO_NAME = 'ver1'
        GIT_REPO_URL = 'https://github.com/Rohitsss-lab/ver1.git'
        PYTHON_PATH = 'C:\\Program Files\\Python313\\python.exe'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout') {
            steps {
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
        stage('Install Dependencies') {
            steps {
                bat 'npm install'
            }
        }
        stage('Run Tests') {
            steps {
                bat 'npm test'
            }
        }
        stage('Bump Version') {
            steps {
                writeFile file: 'bump_version.py', text: '''
import json
import os

def bump(version, bump_type):
    parts = version.split(".")
    major, minor, patch = int(parts[0]), int(parts[1]), int(parts[2])
    if bump_type == "major":
        major += 1; minor = 0; patch = 0
    elif bump_type == "minor":
        minor += 1; patch = 0
    else:
        patch += 1
    return f"{major}.{minor}.{patch}"

bump_type = os.environ.get("BUMP_TYPE", "patch")

with open("versions.json", "r") as f:
    vdata = json.load(f)

old_version = vdata["version"]
new_version = bump(old_version, bump_type)
vdata["version"] = new_version

with open("versions.json", "w") as f:
    json.dump(vdata, f, indent=2)

with open("package.json", "r") as f:
    pdata = json.load(f)

pdata["version"] = new_version

with open("package.json", "w") as f:
    json.dump(pdata, f, indent=2)

with open("NEW_VERSION.txt", "w") as f:
    f.write(new_version)

print(f"Bumped {old_version} -> {new_version}")
'''
                withEnv(["BUMP_TYPE=${params.BUMP_TYPE}"]) {
                    bat '"C:\\Program Files\\Python313\\python.exe" bump_version.py'
                }
            }
        }
        stage('Commit and Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-token',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    script {
                        def newVersion = readFile('NEW_VERSION.txt').trim()
                        env.NEW_VERSION = newVersion
                    }
                    bat '''
                        git config user.email "jenkins@ci.com"
                        git config user.name "Jenkins"
                        git checkout -b release/v%NEW_VERSION%
                        git add versions.json package.json
                        git commit -m "chore: bump version to v%NEW_VERSION%"
                        git remote set-url origin https://oauth2:%GIT_TOKEN%@github.com/YOUR_USERNAME/repo1.git
                        git push origin release/v%NEW_VERSION%
                        git checkout main
                        git merge release/v%NEW_VERSION%
                        git push origin main
                        git tag v%NEW_VERSION%
                        git push origin v%NEW_VERSION%
                    '''
                }
            }
        }
        stage('Deploy') {
            steps {
                echo "Deploying repo1 version ${env.NEW_VERSION}"
                bat 'npm install'
                echo "repo1 deployed successfully on port 3001"
            }
        }
        stage('Notify repo3') {
            steps {
                script {
                    def newVersion = readFile('NEW_VERSION.txt').trim()
                    build job: 'repo3-umbrella',
                          wait: true,
                          parameters: [
                              string(name: 'REPO_NAME', value: 'ver1'),
                              string(name: 'REPO_VERSION', value: newVersion),
                              string(name: 'BUMP_TYPE', value: 'patch')
                          ]
                }
            }
        }
    }
    post {
        success {
            echo "repo1 pipeline completed successfully"
        }
        failure {
            echo "repo1 pipeline FAILED"
        }
    }
}
