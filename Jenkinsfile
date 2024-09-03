pipeline {
    agent any

    environment {
        WORKSPACE_DIR = 'C:\\Jen\\.jenkins\\workspace\\DatatableDemo2'
        MSBUILD_FILE = 'C:\\Jen\\.jenkins\\workspace\\DatatableDemo2\\DatatableDemo.csproj'
        ARTIFACTS_DIR = 'C:\\Artifacts\\DatatableDemo2'
        PACKAGE_LOCATION = 'C:\\Artifacts\\DatatableDemo2\\DatatableDemo.zip'
        DEPLOY_NAME = 'DatatableDemo'
        DEPLOY_PATH = 'https://ec2-52-64-60-183.ap-southeast-2.compute.amazonaws.com:8172/msdeploy.axd'
        GIT_URL = 'https://github.com/binxiong-tunity/DatatableDemo2.git'
    }

    stages {
        stage('Check PR Event') {
            steps {
                script {
                    if (env.CHANGE_ID != null) {
                            env.CONTINUE_PIPELINE = 'true'
                                echo "${env.CONTINUE_PIPELINE}"
                    } else {
                        env.CONTINUE_PIPELINE = 'false'
                    }
                }
            }
        }

        stage('Checkout') {
            when {
                expression {  
                    echo "${env.CONTINUE_PIPELINE}" 
                    return env.CONTINUE_PIPELINE = 'true' 
                } 
            }
            steps {
                // Checkout the code from the repository
                git url: "${GIT_URL}", branch: "${env.CHANGE_BRANCH}"
            }
        }

        stage('Restore') {
            when {
                expression {  
                    echo "${env.CONTINUE_PIPELINE}" 
                    return env.CONTINUE_PIPELINE = 'true' 
                } 
            }
            steps {
                dir(env.WORKSPACE_DIR) {
                    bat 'dotnet restore'
                }
            }
        }

        stage('Build') {
            when {
                expression {  
                    echo "${env.CONTINUE_PIPELINE}" 
                    return env.CONTINUE_PIPELINE = 'true' 
                } 
            }
            steps {
                bat """
                    msbuild ${env.MSBUILD_FILE} /p:Configuration=Release /p:DeployOnBuild=true /p:WebPublishMethod=Package /p:PackageAsSingleFile=true /p:DeleteExistingFiles=true /p:PackageLocation="${env.PACKAGE_LOCATION}" /p:DeployIisAppPath=demo /p:EnvironmentName=Production
                """
            }
        }

        stage('Post-Build') {
            when {
                expression {  
                    echo "${env.CONTINUE_PIPELINE}" 
                    return env.CONTINUE_PIPELINE = 'true' 
                } 
            }
            steps {
                dir(env.ARTIFACTS_DIR) {
                    withCredentials([usernamePassword(credentialsId: 'demo-project-deploy-credentials', usernameVariable: 'DEPLOY_USERNAME', passwordVariable: 'DEPLOY_PASSWORD')]) {
                        bat """
                            .\\${env.DEPLOY_NAME}.deploy.cmd /Y /M:"${env.DEPLOY_PATH}" /U:${env.DEPLOY_USERNAME} /P:${env.DEPLOY_PASSWORD} /A:Basic
                        """
                    }
                    echo 'Post-build steps, e.g., handling artifacts or cleanup'
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
