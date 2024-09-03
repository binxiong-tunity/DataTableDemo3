pipeline {
    agent any

    environment {
        JENKIN_PROJECT_NAME = 'DemoProject2'
        WORKSPACE_BASE_DIR = 'C:\\Jen\\.jenkins\\workspace\\'
        ARTIFACTS_BASEDIR = 'C:\\Artifacts\\'
        PROJECT_NAME = 'DatatableDemo'
        DEPLOY_PATH = 'https://ec2-52-64-60-183.ap-southeast-2.compute.amazonaws.com:8172/msdeploy.axd'
        GIT_URL = 'https://github.com/binxiong-tunity/DatatableDemo2.git'
    }

    stages {
        stage('Check PR Event') {
            steps {
                script {
                    if (env.CHANGE_ID != null) {
                            env.CONTINUE_PIPELINE = 'true'
                            env.WORKSPACE_DIR = "${env.WORKSPACE_BASE_DIR}${env.JENKIN_PROJECT_NAME}_PR-${env.CHANGE_ID}"
                            env.ARTIFACTS_DIR = "${env.ARTIFACTS_BASEDIR}DemoProject2_PR-${env.CHANGE_ID}"
                            env.PACKAGE_LOCATION = "${env.ARTIFACTS_DIR}\\${env.PROJECT_NAME}.zip"
                            env.REPO_NAME = env.GIT_URL.split('/').last().replace('.git', '')

                            echo "Repository Name from Job Name: ${env.REPO_NAME}"
                            env.MSBUILD_FILE = "${env.WORKSPACE_DIR}\\${env.PROJECT_NAME}\\${env.PROJECT_NAME}.csproj"
                            echo "MSBUILD_FILE:${env.MSBUILD_FILE}"
                            echo "CONTINUE_PIPELINE:${env.CONTINUE_PIPELINE}"
                            echo "WORKSPACE_DIR:${env.WORKSPACE_DIR}"
                            echo "ARTIFACTS_DIR:${env.ARTIFACTS_DIR}"
                            echo "PACKAGE_LOCATION:${env.PACKAGE_LOCATION}"
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
                    echo "${env.MSBUILD_FILE}"
                    return env.CONTINUE_PIPELINE = 'true' 
                } 
            }
            steps {
                dir(env.WORKSPACE_DIR) {
                    bat """
                        msbuild ${env.MSBUILD_FILE} /p:Configuration=Release /p:DeployOnBuild=true /p:WebPublishMethod=Package /p:PackageAsSingleFile=true /p:DeleteExistingFiles=true /p:PackageLocation="${env.PACKAGE_LOCATION}" /p:DeployIisAppPath=demo /p:EnvironmentName=Production
                    """
                }
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
