# Jenkins Pipeline Tutorial
In this tutorial, you will get a step by step guide to create Jenkins pipelines for react native mobile app project.

1. [Pipeline Overview](#pipeline-overview)
2. [Continuos Integration](#continuos-integration)
    1. [Install dependencies](#install-dependencies)
    2. [Unit Testing](#unit-testing)
    3. [Code Coverage](#code-coverage)
    4. [Android SDK setup](#android-sdk-setup)
    5. [Generate APK](#generate-apk)
5. [Post build](#post-build)


## Continuos Integration
### Install dependencies
**Prerequisites:**
1. NodeJS Plugin
   - Install "NodeJS Plugin" from Jenkins Plugin Manager
   - Navigate to: Manage Jenkins > Manage Plugins > Available > Search "NodeJS"

2. NodeJS Tool Configuration
   - Navigate to: Manage Jenkins > Tools
   - Scroll to NodeJS installations section
   - Click "Add NodeJS"
   - Configure:
     * Name: `nodejs-16` (or your preferred name)
     * Version: Select required version
     * Save/Apply

Note: The name configured in the NodeJS tool installation will be referenced in your Jenkinsfile's `tools` section. Node.js will be installed automatically when the pipeline executes.

**Syntax:**

```groovy
pipeline {
    agent any
    tools {
        nodejs 'nodejs-16'
    }
    stages{
        stage('Install dependencies'){
            steps{
                sh 'npm install --force--no-audit'
            }
        }
    }
}
```
### Unit Testing
**Note**: this For unit tests with `Jest`

**Prerequisites:**
- `test` command configured in the `package.json` in the `scripts` section: 
    ```json 
        "scripts": {
            "test": "jest"
        }
    ```
- To generate The Test results reports(XML, HTML) You need to:
    1. Add `jest-html-reporter` and `jest-junit` to dependencies list in the package.json file
    2. Add reports config in jest.config.js file : 
    ```js
    reporters: [
        'default',
        ['./node_modules/jest-html-reporter', {
            'pageTitle': 'Jest Test Report',
            'outputPath': 'jest-test-report.html',
            'includeFailureMsg': false
        }],
        ['./node_modules/jest-junit', {
            'suiteName': 'jest tests',
            'outputDirectory': '.',
            'outputName': 'test-results.xml',
            'uniqueOutputName': 'false',
            'classNameTemplate': '{classname}-{title}',
            'titleTemplate': '{classname}-{title}',
            'ancestorSeparator': ' › ',
            'usePathForSuiteName': 'true'
        }]
    ]
    ```
The HTML file name specified in the `outputPath` in your `Jest configuration` will be used in the `publishHTML` step in the post-build actions

**Syntax**
```groovy
stage('Unit Testing'){
    steps{
        sh 'npm test -- --u'
    }
}
```
### Code Coverage
**Prerequisites:**
- `Coverage` command configured in the `package.json` in the `scripts` section: 
    ```json 
        "scripts": {
            "coverage": "jest --coverage"
        }
    ```
**Syntax**
```groovy
stage('Code Coverage'){
    steps{
        catchError(buildResult: 'SUCCESS', message: 'Coverage', stageResult: 'UNSTABLE') {
            sh 'npm run coverage'
        }
    }
}
```
- The `CatchError` block will prevent the pipeline from failing if the `sh 'npm run coverage'` command fails.
- A Folder named `coverage` will be created in the workspace and the coverage report will be saved in it.
<p align="center">
    <img src="image-8.png" width="350" height="170" alt="HTML Publisher Plugin"/>
</p>

- The Coverage file report will be saved in the `lcov-report` folder under the name `index.html`.

### Android SDK setup
```groovy
environment {
    ANDROID_SDK_ROOT = '/var/jenkins_home/android-sdk'
    ANDROID_HOME = '/var/jenkins_home/android-sdk'
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
```
### Generate APK
```groovy
environment {
    CURRENT_DATE = new Date().format('dd_MM_yyyy')
    APK_NAME = "HippoShopping-${CURRENT_DATE}.apk"
}

stage('Generate APK') {
    environment {
        APK_ARCHIVE_PATH = "android/app/build/outputs/apk/release/${APK_NAME}"
    }
    steps {
        dir('android') {
            sh './gradlew app:assembleRelease --stacktrace'

            sh '''
                cd app/build/outputs/apk/release
                mv app-release.apk ${APK_NAME}
            '''
        }
    }
    post {
        success {
            archiveArtifacts artifacts: env.APK_ARCHIVE_PATH, fingerprint: true
        }
    }
}
```
## Post build
![alt text](image-1.png)
**Prerequisites:**
1. HTML Publisher and JUnit plugins
   - Install "HTML Publisher" and "JUnit" from Jenkins Plugin Manager
   - Navigate to: Manage Jenkins > Manage Plugins > Available > Search "HTML Publisher" and "JUnit"
<img src="image-7.png" width="400" height="170" alt="HTML Publisher Plugin"/>
<img src="image-9.png" width="400" height="130" alt="Junit Plugin"/>

**Syntax:**
```groovy
post{
    always {
        junit allowEmptyResults: true, stdioRetention: '', testResults: 'test-results.xml'                   
        publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: '', reportFiles: 'jest-test-report.html', reportName: 'Jest TestHTML Report', reportTitles: '', useWrapperFileDirectly: true])

        // Publish Code Coverage Report
        publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: './coverage/lcov-report', reportFiles: 'index.html', reportName: 'Code Coverage HTML Report', reportTitles: '', useWrapperFileDirectly: true])
    }  
}
```
**Note:** syntax From "Pipeline Syntax" ->  "Snippet Generator" -> `publishHTML: Publish HTML reports`





