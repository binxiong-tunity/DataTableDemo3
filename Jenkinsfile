pipeline {
    agent any

    stages {
        stage('Determine Pipeline Type') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                        if (env.CHANGE_ID) {
                            // If it's a PR
                            def targetBranch = sh(script: 'curl -s -H "Authorization: token ${GITHUB_TOKEN}" https://api.github.com/repos/${GITHUB_REPO}/pulls/${env.CHANGE_ID} | jq -r .base.ref', returnStdout: true).trim()
                            
                            if (targetBranch ==~ /main/) {
                                echo "Loading Jenkinsfile-CI for PR to main branch"
                                load 'Jenkinsfile-CI'
                            } else {
                                error "Target branch ${targetBranch} is not recognized."
                            }
                        } else {
                            // If it's not a PR, check if the branch is main
                            if (env.BRANCH_NAME ==~ /stable/*) {
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
