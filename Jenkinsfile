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
        LOCAL_ARCHIVE_DIR = "C:\\Artifacts\\${PROJECT_NAME}"
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
                            } if (targetBranch ==~  /^(Stable\/V\d+)$/ ) else {
                                currentBuild.description = "PR to Stable"
                               env.PIPELINE_TYPE = 'CI-Light'
                            } else {
                              error "Target Branch ${env.targetBranch} is not recognized."
                            }
                        } else {
                            if (env.BRANCH_NAME && env.BRANCH_NAME ==~ /^v\d+\.\d+\.\d+$/) {
                                echo "Tag detected: ${env.GIT_TAG}"
                                env.PIPELINE_TYPE = 'CD'
                                currentBuild.description = "Tag build: ${env.GIT_TAG}"
                            } 
                            else {
                                error "Branch ${env.BRANCH_NAME} is not recognized."
                            }
                        }
                    }
                }
            }
        }
        
        stage('CI Pipeline Stages') {
            when {
                expression { return env.CONTINUE_PIPELINE == 'true' || env.PIPELINE_TYPE == 'CI-Light' }
            }
            stages {
                stage('Check PR Event') {
                    steps {
                        script {
                           
                            if (env.CHANGE_ID != null) {
                                echo 'Check PR Event'
                                env.CONTINUE_PIPELINE = 'true'
                                env.WORKSPACE_DIR = "${env.WORKSPACE_BASE_DIR}${env.JENKIN_PROJECT_NAME}_PR-${env.CHANGE_ID}"
                                env.ARTIFACTS_DIR = "${env.WORKSPACE_DIR}\\Artifacts"
                                env.PACKAGE_LOCATION = "${env.ARTIFACTS_DIR}\\${env.PROJECT_NAME}.zip"
                                env.REPO_NAME = env.GIT_URL.split('/').last().replace('.git', '')
                                env.COMMIT_ID = powershell(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                                echo "Commit ID: ${env.COMMIT_ID}"
                                env.ZIP_FILE = "${env.PROJECT_NAME}__${env.COMMIT_ID}-SNAPSHOT.zip" 
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
                        expression {return env.CONTINUE_PIPELINE == 'true' || env.PIPELINE_TYPE == 'CI-Light' }
                    }
                    steps {
                        // Checkout the code from the repository
                        echo 'Checkout the code from the repository'
                        git url: "${env.GIT_URL}", branch: "${env.CHANGE_BRANCH}"
                    }
                }

                stage('Restore') {
                    when {
                        expression { return env.CONTINUE_PIPELINE == 'true' || env.PIPELINE_TYPE == 'CI-Light' }
                    }
                    steps {
                        dir(env.WORKSPACE_DIR) {
                            bat 'dotnet restore'
                        }
                    }
                }

                stage('Build') {
                    when {
                        expression { return env.CONTINUE_PIPELINE == 'true' || env.PIPELINE_TYPE == 'CI-Light' }
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
                        expression { return env.CONTINUE_PIPELINE == 'true' || env.PIPELINE_TYPE == 'CI-Light' }
                    }
                    steps {
                        echo 'Testing steps'
                    }
                }

                stage('QA') {
                    when {
                        expression { return env.CONTINUE_PIPELINE == 'true' || env.PIPELINE_TYPE == 'CI-Light' }
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
                                 // Archive artifacts 
                                    script {
                                    // Create a zip file with the dynamic name
                                    bat """
                                        powershell Compress-Archive -Path "${env.ARTIFACTS_DIR}\\*" -DestinationPath ${env.ZIP_FILE}
                                    """
        
                                    // Archive the zip file
                                    archiveArtifacts artifacts: "${env.ZIP_FILE}", allowEmptyArchive: false
                                }
                        }
                    }
                }

                stage('Transfer Artifacts') {
                    steps {
                        script {
                            echo 'Transfer Artifacts to Local Archive Directory'
                
                            // Use PowerShell to clean up, then transfer and replace the zip file in the local archive directory
                            powershell """
                                # Ensure the destination directory exists
                                if (-Not (Test-Path -Path "${env.LOCAL_ARCHIVE_DIR}")) {
                                    New-Item -Path "${env.LOCAL_ARCHIVE_DIR}" -ItemType Directory
                                } else {
                                     # Remove only the specific ZIP file from the destination directory
                                    \$zipFilePath = "${env.LOCAL_ARCHIVE_DIR}\\${env.ZIP_FILE}"
                                    if (Test-Path -Path \$zipFilePath) {
                                        Remove-Item -Path \$zipFilePath -Force
                                    }
                                }
                
                                # Transfer the zip file to the local archive directory
                                Copy-Item -Path "${env.ZIP_FILE}" -Destination "${env.LOCAL_ARCHIVE_DIR}" -Force

                            """
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
                stage('Setup Environment') {
                    steps {
                             script {
                                env.CONTINUE_PIPELINE = 'true'
                                env.WORKSPACE_DIR = "${env.WORKSPACE_BASE_DIR}${env.JENKIN_PROJECT_NAME}_TAG-${env.BRANCH_NAME}"
                                 env.ARTIFACTS_DIR = "${env.WORKSPACE_DIR}\\Artifacts"
                                env.PACKAGE_LOCATION = "${env.ARTIFACTS_DIR}\\${env.PROJECT_NAME}.zip"
                                env.REPO_NAME = env.GIT_URL.split('/').last().replace('.git', '')
                                env.COMMIT_ID = powershell(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                                echo "Commit ID: ${env.COMMIT_ID}"
                                env.ZIP_FILE = "${env.PROJECT_NAME}__${env.COMMIT_ID}-SNAPSHOT.zip" 
                                echo "Repository Name from Job Name: ${env.REPO_NAME}"

                                echo "CONTINUE_PIPELINE:${env.CONTINUE_PIPELINE}"
                                echo "WORKSPACE_DIR:${env.WORKSPACE_DIR}"
                                echo "ARTIFACTS_DIR:${env.ARTIFACTS_DIR}"
                                echo "PACKAGE_LOCATION:${env.PACKAGE_LOCATION}"
                             }
                    }
                }
                stage('Retrieving Artifacts') {
                    steps {
                        
                         script {
                             echo "PACKAGE_LOCATION == ${env.PACKAGE_LOCATION}"
                            echo 'Retrieve and Extract Artifacts from the Jenkins server'
            
                            // Define the PowerShell script as a string
                            def powershellScript = """
                                # Define paths
                                \$destinationPath = '${env.ARTIFACTS_DIR}'
                                \$destinationZipPath = '${env.ARTIFACTS_DIR}\\${env.PROJECT_NAME}__${env.COMMIT_ID}-SNAPSHOT.zip'
                                \$sourceZipPath = '${env.LOCAL_ARCHIVE_DIR}\\${env.PROJECT_NAME}__${env.COMMIT_ID}-SNAPSHOT.zip'
                
                                # Ensure destination directory exists
                                if (-Not (Test-Path -Path \$destinationPath)) {
                                    New-Item -Path \$destinationPath -ItemType Directory
                                }
                
                                # Copy the zip file to the local archive directory
                                Copy-Item -Path \$sourceZipPath -Destination \$destinationPath -Force
                
                                # Define the path of the extracted folder
                                \$extractPath = Join-Path -Path \$destinationPath -ChildPath '${env.PROJECT_NAME}'
                
                                # Unzip the file
                                Expand-Archive -Path (Join-Path -Path \$destinationPath -ChildPath (Split-Path -Path \$sourceZipPath -Leaf)) -DestinationPath \$extractPath -Force

                                 # Delete the zip file
                                 Remove-Item -Path \$destinationZipPath -Force
                            """
                
                            // Execute the PowerShell script
                            powershell(script: powershellScript)
                        }
                    }
                }

                stage('Deploy to Staging') {
                     when {
                        expression {  
                            echo "${env.CONTINUE_PIPELINE}" 
                            return env.CONTINUE_PIPELINE = 'true' 
                        } 
                    }
                    steps {
                        dir("${env.ARTIFACTS_DIR}\\${env.PROJECT_NAME}") {
                            withCredentials([usernamePassword(credentialsId: 'demo-project-deploy-credentials', usernameVariable: 'DEPLOY_USERNAME', passwordVariable: 'DEPLOY_PASSWORD')]) {
                                bat """
                                    .\\${env.PROJECT_NAME}.deploy.cmd /Y /M:"${env.DEPLOY_PATH}" /U:"${DEPLOY_USERNAME}" /P:"$DEPLOY_PASSWORD" /A:Basic
                                """
                            }
                            echo 'Post-build steps, e.g., handling artifacts or cleanup'
                        }
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
