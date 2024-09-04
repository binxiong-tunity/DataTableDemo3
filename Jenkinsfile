pipeline {
    agent any

    environment {
        GITHUB_REPO = 'https://github.com/binxiong-tunity/DatatableDemo3.git'
    }
    
    stages {
        stage('Determine Pipeline Type') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                        if (env.CHANGE_ID) {
                            // If it's a PR
                            def response = sh(script: 'curl -s -H "Authorization: token ${GITHUB_TOKEN}" https://api.github.com/repos/${GITHUB_REPO}/pulls/${env.CHANGE_ID}', returnStdout: true).trim()
                            def targetBranch = powershell(returnStdout: true, script: """
                            \$response = '${response}' | ConvertFrom-Json
                            \$response.base.ref
                            """).trim()
                            
                            if (targetBranch ==~ /main/) {
                                echo "Loading Jenkinsfile-CI for PR to main branch"
                                load 'Jenkinsfile-CI'
                            } else {
                                error "Target branch ${targetBranch} is not recognized."
                            }
                        } else {
                            // If it's not a PR, check if the branch is main
                            if (targetBranch ==~ /Stable\/.*/) {
                                echo "Loading Jenkinsfile-CD for main branch"
                                load 'Jenkinsfile-CD'
                            } 
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
