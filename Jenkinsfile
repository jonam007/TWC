pipeline {
    agent any

    environment {
        // Environment variables
        ANDROID_HOME = "/home/ec2-user/Android/Sdk"
        PATH = "${ANDROID_HOME}/platform-tools:${ANDROID_HOME}/tools:${ANDROID_HOME}/tools/bin:${PATH}"
        // Add Node.js to PATH if installed via nvm
        PATH = "/home/ec2-user/.nvm/versions/node/v20.x/bin:${PATH}"
        
        // AWS credentials (better to use Jenkins credentials)
        AWS_ACCESS_KEY_ID = credentials('aws-access-key')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
        AWS_DEFAULT_REGION = 'ap-south-1'
        
        // Build config
        S3_BUCKET = 'mysthreebuck'
        APK_PATH = 'android/app/build/outputs/apk/release/app-release.apk'
    }

    tools {
        nodejs 'NodeJS-LTS'  // Requires NodeJS plugin installed
    }

    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: 'main']],
                    userRemoteConfigs: [[url: 'https://github.com/jonam007/TWC.git']],
                    extensions: [[
                        $class: 'CleanBeforeCheckout'
                    ]]
                ])
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'  // Clean install, better for CI than npm install
                sh 'npx pod-install'  // For iOS if needed
            }
        }

        stage('Build Android Release APK') {
            steps {
                dir('android') {
                    sh './gradlew assembleRelease --stacktrace --info'
                }
            }
        }

        stage('Archive APK') {
            steps {
                archiveArtifacts artifacts: "${APK_PATH}", fingerprint: true
                junit '**/test-results/**/*.xml'  // Archive test results if available
            }
        }

        stage('Upload to S3') {
            steps {
                withAWS(credentials: 'aws-credentials') {  // Better than env vars
                    s3Upload(
                        file: "${APK_PATH}",
                        bucket: "${S3_BUCKET}",
                        path: "app-${BUILD_NUMBER}.apk"  // Unique filename per build
                    )
                }
            }
        }
    }

    post {
        always {
            cleanWs()  // Clean workspace after build
            script {
                currentBuild.description = "Build #${BUILD_NUMBER}"
            }
        }
        success {
            slackSend color: 'good', message: "Build ${BUILD_NUMBER} succeeded!"
        }
        failure {
            slackSend color: 'danger', message: "Build ${BUILD_NUMBER} failed!"
        }
    }
}
