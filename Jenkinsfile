pipeline {
    agent any

    environment {
        GITHUB_REPO = 'binxiong-tunity/DatatableDemo3'
        JENKIN_PROJECT_NAME = 'DemoProject3'
        WORKSPACE_BASE_DIR = 'C:\\Jen\\.jenkins\\workspace\\'
        ARTIFACTS_BASEDIR = 'C:\\Artifacts\\'
        PROJECT_NAME = 'DatatableDemo'
        DEPLOY_PATH = 'https://ec2-52-64-60-183.ap-southeast-2.compute.amazonaws.com:8172/msdeploy.axd'
        GIT_URL = 'https://github.com/binxiong-tunity/DatatableDemo3.git'

    }
    
    stages {
        stage('Determine Pipeline Type') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                        if (env.CHANGE_ID) {
                            // If it's a PR
                            def targetBranch = powershell(returnStdout: true, script: """
                                \$headers = @{ Authorization = 'token ${GITHUB_TOKEN}' }
                                \$response = Invoke-RestMethod -Uri "https://api.github.com/repos/${GITHUB_REPO}/pulls/${env.CHANGE_ID}" -Headers \$headers
                                \$response.base.ref
                                """).trim()
                            
                            if (targetBranch ==~ /main/) {
                                echo "PR target branch is main. Proceeding with CI pipeline."
                                currentBuild.description = "PR to main"
                                env.PIPELINE_TYPE = 'CI'
                            } else {
                                error "Target branch ${targetBranch} is not recognized."
                            }
                        } else {
                            // Not a PR, check if the branch is stable
                            if (env.BRANCH_NAME ==~ /Stable\/.*/) {
                                echo "Branch is stable. Proceeding with CD pipeline."
                                currentBuild.description = "Stable branch"
                                env.PIPELINE_TYPE = 'CD'
                            } else {
                                error "Branch ${env.BRANCH_NAME} is not recognized."
                            }
                        }
                    }
                }
            }
        }
        
        stage('CI Pipeline Stages') {
            when {
                expression { return env.PIPELINE_TYPE == 'CI' }
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
                                env.COMMIT_ID = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                                env.ZIP_FILE = "${env.PROJECT_NAME}__${env.COMMIT_ID}-SNAPSHOT.ZIP" 
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
                        expression { return env.CONTINUE_PIPELINE == 'true' }
                    }
                    steps {
                        // Checkout the code from the repository
                        git url: "${GIT_URL}", branch: "${env.CHANGE_BRANCH}"
                    }
                }

                stage('Restore') {
                    when {
                        expression { return env.CONTINUE_PIPELINE == 'true' }
                    }
                    steps {
                        dir(env.WORKSPACE_DIR) {
                            bat 'dotnet restore'
                        }
                    }
                }

                stage('Build') {
                    when {
                        expression { return env.CONTINUE_PIPELINE == 'true' }
                    }
                    steps {
                        dir(env.WORKSPACE_DIR) {
                            bat """
                                msbuild ${env.MSBUILD_FILE} /p:Configuration=Release /p:DeployOnBuild=true /p:WebPublishMethod=Package /p:PackageAsSingleFile=true /p:DeleteExistingFiles=true /p:PackageLocation="${env.PACKAGE_LOCATION}" /p:DeployIisAppPath=demo /p:EnvironmentName=Production
                            """
                        }
                    }
                }

                stage('Testing') {
                    when {
                        expression { return env.CONTINUE_PIPELINE == 'true' }
                    }
                    steps {
                        echo 'Testing steps'
                    }
                }

                stage('QA') {
                    when {
                        expression { return env.CONTINUE_PIPELINE == 'true' }
                    }
                    steps {
                        echo 'QA steps - run SonarCube'
                    }
                }

                stage('Post-Build') {
                    when {
                        expression { return env.CONTINUE_PIPELINE == 'true' }
                    }
                    steps {
                          script {
                            // Create a zip file with the dynamic name
                            bat """
                                powershell Compress-Archive -Path ${env.ARTIFACTS_DIR} -DestinationPath ${env.ZIP_FILE}
                            """
                            
                            // Archive the zip file
                            archiveArtifacts artifacts: "${env.ZIP_FILE}", allowEmptyArchive: false
                        }
                    }
                }
            }
        }
        
        stage('CD Pipeline Stages') {
            when {
                expression { return env.PIPELINE_TYPE == 'CD' }
            }
            stages {
                stage('Retrieving Artifacts') {
                    steps {
                        echo 'Retrieve Artifacts from the Jenkins server'
                    }
                }

                stage('Deploy to Staging') {
                    steps {
                        echo 'Deployment steps to staging server'
                    }
                }

                stage('Approval to Production') {
                    steps {
                        echo 'Approval to Production'
                        input message: 'Approve Deployment to Production?', ok: 'Deploy'
                    }
                }

                stage('Deploy to Production') {
                    steps {
                        echo 'Deployment steps to production server'
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
