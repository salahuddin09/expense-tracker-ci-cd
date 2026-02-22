// This is the Jenkinsfile that will be used to build & test the project.
pipeline {
    agent any
    options {
        skipDefaultCheckout()
        // Add retry for non-resumable steps
        retry(2)
        timeout(time: 60, unit: 'MINUTES')  // Overall pipeline timeout
    }
   // tools {
   //     maven "mvn"
   //     nodejs "node"
  //  }

    environment {
        RENDER_API_KEY = credentials('render-api-key')
        RENDER_BACKEND_SERVICE_ID = 'srv-d6cnl8ogjchc739ot3hg'
        RENDER_BACKEND_DEPLOY_HOOK = "https://api.render.com/deploy/${RENDER_BACKEND_SERVICE_ID}?key=fn4I37yBx3w"
        RENDER_FRONTEND_SERVICE_ID = 'srv-d6coq595pdvs739k8sb0'
        RENDER_FRONTEND_DEPLOY_HOOK = "https://api.render.com/deploy/${RENDER_FRONTEND_SERVICE_ID}?key=-qOQl_RmQ4M"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', credentialsId: 'Git token', url: 'https://github.com/salahuddin09/expense-tracker-ci-cd.git'
            }
        }
        stage('Build') {
            parallel {
                stage('Java') {
                    steps {
                       // dir('expense-tracker-service') {
                       //     sh 'mvn clean install'
                       // }
                        dir('expense-tracker-service') {
                            // Use bat for Windows, or wrap with isUnix() for cross-platform
                            bat 'mvn clean install -DskipTests'
                        }
                    }
                }

                stage('Angular') {
                    steps {
                      //  dir('expense-tracker-ui') {
                      //      sh 'npm install'
                      //      sh './node_modules/.bin/ng build --configuration production'
                      //  }
                        dir('expense-tracker-ui') {
                            bat 'npm install'
                            bat 'node_modules\\.bin\\ng build --configuration production'
                        }
                    }
                }
            }
        }

        stage('Test') {
            steps {
              //  script {
              //      sh 'cd expense-tracker-service && mvn test'
              //  }
                dir('expense-tracker-service') {
                    bat 'mvn test'
                }
            }
        }
        stage('Sonar') {
            steps {
                dir('expense-tracker-service') {
                    withSonarQubeEnv('MySonarServer') {
                        bat 'mvn sonar:sonar'  // ✅ Use bat for Windows
                    }
                }
            }
            // ✅ post must be at STAGE level (same indentation as 'steps')
            post {
                success {
                    script {
                        timeout(time: 10, unit: 'MINUTES') {
                            def qualityGate = waitForQualityGate()
                            if (qualityGate.status != 'OK') {
                                error "SonarQube Quality Gate failed: ${qualityGate.status}"
                            } else {
                                echo "✅ SonarQube analysis passed."
                            }
                        }
                    }
                }
                failure {
                    echo "❌ SonarQube analysis failed during execution."
                }
            }
        }
        stage('Deploy to Render') {
            steps {
                script {
                    // ✅ Get changed files - Windows compatible
                    def changedFiles
                    try {
                        // Use bat for Windows
                        changedFiles = bat(script: 'git diff --name-only HEAD HEAD~1', returnStdout: true)?.trim()
                    } catch (Exception e) {
                        echo "⚠️ Could not diff (possibly initial commit): ${e.message}"
                        changedFiles = bat(script: 'git ls-files', returnStdout: true)?.trim()
                    }

                    // ✅ Safe split - handles empty string
                    def filesList = changedFiles ? changedFiles.split('\n').collect { it.trim() }.findAll { it } : []
                    echo "📄 Changed files (${filesList.size()}):\n${filesList.join('\n')}"

                    // ✅ Check for backend changes
                    def backendChanged = filesList.any { file ->
                        file.startsWith("expense-tracker-service/") ||
                                file == "Dockerfile" ||
                                file == "Jenkinsfile" ||
                                file.startsWith("pom.xml")
                    }

                    // ✅ Check for frontend changes
                    def frontendChanged = filesList.any { file ->
                        file.startsWith("expense-tracker-ui/") ||
                                file == "Dockerfile" ||
                                file == "Jenkinsfile" ||
                                file.startsWith("package.json")
                    }

                    // ✅ Deploy backend if needed
                    if(backendChanged) {
                        echo "🚀 Backend changes detected. Deploying..."
                        def backendResponse = httpRequest(
                                url: "${RENDER_BACKEND_DEPLOY_HOOK}",
                                httpMode: 'POST',
                                validResponseCodes: '200:299'
                        )
                        echo "✅ Backend deploy response: ${backendResponse}"
                    } else {
                        echo "⏭️ No backend changes. Skipping."
                    }

                    // ✅ Deploy frontend if needed
                    if(frontendChanged) {
                        echo "🚀 Frontend changes detected. Deploying..."
                        def frontendResponse = httpRequest(
                                url: "${RENDER_FRONTEND_DEPLOY_HOOK}",
                                httpMode: 'POST',
                                validResponseCodes: '200:299'
                        )
                        echo "✅ Frontend deploy response: ${frontendResponse}"
                    } else {
                        echo "⏭️ No frontend changes. Skipping."
                    }
                }
            }
        }
    }
    post {
        always {
            // Clean up workspace to save disk space
            cleanWs(notFailBuild: true)
        }
        success {
            echo '✅ Build was successful!'
        }
        failure {
            echo '❌ Build failed. Check logs above.'
            // Optional: Send notification here
        }
    }
}