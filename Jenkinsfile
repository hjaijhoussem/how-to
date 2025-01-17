pipeline {
    agent any
    tools {
        nodejs 'nodejs'
    }
    
    environment {
        // Android SDK paths
        ANDROID_SDK_ROOT = '/var/jenkins_home/android-sdk'
        ANDROID_HOME = '/var/jenkins_home/android-sdk'
        
        // APK path configuration
        APK_OUTPUT_PATH = 'android/app/build/outputs/apk/release'
        APK_FILE_NAME = 'app-release.apk'
        
        // Nexus Configuration
        NEXUS_VERSION = 'nexus3'
        NEXUS_PROTOCOL = 'http'
        NEXUS_URL = 'your-nexus-url:8081'
        NEXUS_REPOSITORY = 'android-releases'
        NEXUS_CREDENTIAL_ID = 'nexus-credentials'
        
        // Artifact details
        ARTIFACT_VERSION = "${BUILD_NUMBER}"
        GROUP_ID = 'com.youcompany.app'
        ARTIFACT_ID = 'android-app'
    }

    stages {
        stage('Checkout code'){
            steps{
                checkout scmGit(branches: [[name: '*/jenkins-test']], userRemoteConfigs: [[credentialsId: 'jenkins-ssh', url: 'git@github.com:hippo-labs-inc/HippoShopping.git']])
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh 'npm install --force --no-audit '
            }
        }

        stage('Setup Android SDK') {
            steps {
                script {
                    try {
                        sh '''
                            # Install necessary tools
                            apt-get update
                            apt-get install -y wget unzip

                            # Create Android SDK directory if it doesn't exist
                            mkdir -p $ANDROID_SDK_ROOT
                            
                            # Download and install command line tools if not exists
                            if [ ! -d "$ANDROID_SDK_ROOT/cmdline-tools/latest" ]; then
                                echo "Installing Android SDK command line tools..."
                                cd $ANDROID_SDK_ROOT
                                
                                # Clean up any existing files
                                rm -f commandlinetools-linux-9477386_latest.zip*
                                rm -rf cmdline-tools
                                
                                wget https://dl.google.com/android/repository/commandlinetools-linux-9477386_latest.zip
                                unzip -o -q commandlinetools-linux-9477386_latest.zip
                                
                                # Create the proper directory structure
                                mkdir -p cmdline-tools/latest
                                mv cmdline-tools/* cmdline-tools/latest/ 2>/dev/null || true
                                rm -rf cmdline-tools/latest/cmdline-tools
                                rm -f commandlinetools-linux-9477386_latest.zip
                            fi
                            
                            export PATH=$ANDROID_SDK_ROOT/cmdline-tools/latest/bin:$ANDROID_SDK_ROOT/platform-tools:$PATH
                            
                            # Accept licenses
                            yes | sdkmanager --licenses
                            
                            # Install required SDK components
                            sdkmanager --update
                            sdkmanager \
                                "platform-tools" \
                                "platforms;android-33" \
                                "build-tools;33.0.0" \
                                "extras;android;m2repository" \
                                "extras;google;m2repository"

                            # Create local.properties in the android directory
                            cd ${WORKSPACE}
                            mkdir -p android
                            echo "sdk.dir=$ANDROID_SDK_ROOT" > android/local.properties
                            
                            # Make gradlew executable
                            chmod +x android/gradlew || true
                        '''
                    } catch (Exception e) {
                        echo "Failed to setup Android SDK: ${e.message}"
                        currentBuild.result = 'FAILURE'
                        error("SDK setup failed")
                    }
                }
            }
        }
        
        stage('Generate APK') {
            steps {
                dir('android') {
                    sh './gradlew app:assembleRelease --stacktrace'
                }
            }
            post {
                success {
                    archiveArtifacts artifacts: "${APK_OUTPUT_PATH}/*.apk", fingerprint: true
                }
            }
        }

        stage('Publish to Nexus') {
            steps {
                script {
                    def apkFile = "${APK_OUTPUT_PATH}/${APK_FILE_NAME}"
                    
                    if (!fileExists(apkFile)) {
                        error "APK file not found: ${apkFile}"
                    }

                    // Using Nexus Upload Plugin
                    nexusArtifactUploader(
                        nexusVersion: NEXUS_VERSION,
                        protocol: NEXUS_PROTOCOL,
                        nexusUrl: NEXUS_URL,
                        groupId: GROUP_ID,
                        version: ARTIFACT_VERSION,
                        repository: NEXUS_REPOSITORY,
                        credentialsId: NEXUS_CREDENTIAL_ID,
                        artifacts: [
                            [artifactId: ARTIFACT_ID,
                             classifier: '',
                             file: apkFile,
                             type: 'apk']
                        ]
                    )
                    
                    echo """
                    APK published to Nexus:
                    ${NEXUS_PROTOCOL}://${NEXUS_URL}/repository/${NEXUS_REPOSITORY}/${GROUP_ID.replace('.', '/')}/${ARTIFACT_ID}/${ARTIFACT_VERSION}/${ARTIFACT_ID}-${ARTIFACT_VERSION}.apk
                    """
                }
            }
        }
    }
    
    post {
        always {
            junit allowEmptyResults: true, stdioRetention: '', testResults: 'test-results.xml'                   
            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: '', reportFiles: 'jest-test-report.html', reportName: 'Jest TestHTML Report', reportTitles: '', useWrapperFileDirectly: true])
        }
        failure {
            echo 'Build failed!'
        }
        success {
            echo 'Build successful!'
        }
    }
} 