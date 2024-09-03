pipeline {
    agent any

    stages {
        stage('Determine Pipeline Type') {
            steps {
                script {
                    if (env.BRANCH_NAME ==~ /PR-.*/) {
                        load 'Jenkinsfile-CI'
                    } else if (env.BRANCH_NAME ==~ /refs\/tags\/.*/) {
                        load 'Jenkinsfile-CD'
                    } else {
                        error "Unknown branch type: ${env.BRANCH_NAME}"
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
