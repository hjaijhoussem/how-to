# Jenkins Pipeline Tutorial

## Pipeline Overview 
![alt text](image-5.png)

## Pipeline Syntax
```groovy
pipeline{


    environment{
        // Instead of using the Username and password credentials, use the Secret text 
        DOCKER_USERNAME = credentials('docker_username') // Secret text
        DOCKER_PASSWORD = credentials('docker_password') // Secret text
    }

    options{
        disableResume()  // Prevents the pipeline from resuming after a Jenkins restart
        disableConcurrentBuilds abortPrevious: true // Abort running builds when a new build is triggered
    }
}
```

## Continuos Integration
![alt text](image.png)
### Buil/Install dependencies
#### Node.js Project
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
                sh 'npm install --no-audit'
            }
        }
    }
}
```
### Dependency Check
#### Node.js Project
**Prerequisites:**
1. OWASP Dependency-Check Plugin
   - Install "NodeJS Plugin" from Jenkins Plugin Manager
   - Navigate to: Manage Jenkins > Manage Plugins > Available > Search "OWASP Dependency-Check"

<p align="center">
    <img src="image-6.png" width="400" height="170" alt="OWASP Dependency Check Plugin"/>
</p>

2. Dependency-check Tool Configuration
   - Navigate to: Manage Jenkins > Tools
   - Scroll to Dependecy-Check installations section
   - Click "Add"
   - Configure:
     * Name: `owasp-10.0.3` (or your preferred name)
     * Mark "Install automatically"
     * Click "Add Installer"
     * Select "Install from github"
     * Version: Select required version
     * Save/Apply

**Syntax:**
```groovy
stage('Dependency Scanninig'){
    parallel{

        stage('NPM Dependency Audit'){
            steps{
                // Fail the build when a critical vunerability exist
                sh 'npm audit --audit-level=critical' 
            }
        }

        stage('OWASP Dependency Check'){
            steps{
                dependencyCheck additionalArguments: '''
                    --scan \'./\'  
                    --out \'./\' 
                    --format \'ALL\' 
                    --prettyPrint''', odcInstallation: 'owasp-10.0.3'

                // Fail the build if one of The vunerability threshold is excided, and generate a report
                dependencyCheckPublisher failedTotalCritical: 1, failedTotalHigh: 4, failedTotalLow: 90, failedTotalMedium: 8, pattern: 'dependency-check-report.xml', stopBuild: true
            }
        }
    }
}
```
OWASP arguments:
- `--scan`: code folder path
- `--out`: the folder where the output files will generated
- `--format`: Output format(XML, JSON...)
- `--prettyPrint`: print the output in the console log 
- `odcInstallation`: dependecy check tool name

You can find this syntax in the **pipeline syntax generator**, search for **dependencyCheck Invoke Dependency-Check** and **dependencyCheckPublisher: Publish Dependency-Check results** 

Note: The **OWASP Dependency Check** will take too much time (~28 minutes). This can be avoided if you provide a dependency database key.
### Unit Testing
### Code Coverage
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
## Continuos Deployment
![alt text](image-2.png)
## Continuos Delivery
![alt text](image-4.png)
## Post build
![alt text](image-1.png)
**Prerequisites:**
1. HTML Publisher Plugin
   - Install "HTML Publisher" from Jenkins Plugin Manager
   - Navigate to: Manage Jenkins > Manage Plugins > Available > Search "HTML Publisher"
**Syntax:**
<img src="image-7.png" width="400" height="170" alt="HTML Publisher Plugin"/>
```groovy
post{
    
    // Publish Dependency Check Reports
    junit allowEmptyResults: true, stdioRetention: '', testResults: 'dependency-check-report.xml'

    publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: './', reportFiles: 'dependency-check-jenkins.html', reportName: 'Dependency Check HTML Report', reportTitles: '', useWrapperFileDirectly: true])

    // Publish Junit XML Report
    junit allowEmptyResults: true, stdioRetention: '', testResults: 'test-results.xml'

    // Publish Code Coverage Report
    publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: './coverage/lcov-report', reportFiles: 'index.html', reportName: 'Code Coverage HTML Report', reportTitles: '', useWrapperFileDirectly: true])

    
}
```
**Note:** syntax From "Pipeline Syntax" ->  "Snippet Generator" -> `publishHTML: Publish HTML reports`
 


