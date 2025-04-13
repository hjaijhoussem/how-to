# React Js Workflow 

1. [Workfow stages overview](#workfows-stages-overview)
2. [kubernetes and GitOPS](#kubernetes-and-gitops)


## Workfow stages overview
**Note:** Usually A pipeline does not implement all this practices, but   
  <p align="center">
      <img src="../images/global-pipe-overview-kodekloud.png" alt="Global Pipeline Overview - Kodekloud"/>
  </p>

### Continuos Integration
1. Build/install Dependencies
2. OWASP Dependency Check  and NPM Dependency Audits
3. Unit tests
4. Code Coverage
5. SAST (sonarqube)
6. Quality Gates (sonarqube)
7. Building docker image
8. Vulnerability scan (Trivy)
9. Push image to registry
### Continuos Deployment
1. AWS EC2 VM Deploy
2. Integration Testing
3. Raise PR to main branch
### Contiuos Delivery
1. Update Docker image Tags (Triggered by the PR raised on main)
2. Deploy to Kubernetes (GitOps - ArgoCD) (Triggered by the PR raised on main)
3. Dynamic Application Security Testion DAST (OWASP ZAP) (Triggered by the PR raised on main), If passed, Merge The PR to main
4. Approval Stage
5. AWS Lambda Deploy, Update Lambda Configurations
6. AWS Lambda Invocation/Testing
### Post Build
**Archiving Reports:**
- Unit Test Reports
- Coverage Reports
- Vulnerability Reports
- Dependency Scan Reports

**Publishing Reports:**
- Publish to Jenkins
- Publish to AWS S3

**Slack Notifications for All States and Stages**

## kubernetes and GitOPS
### K8s basics 

Kubernetes (K8s) is a container orchestration platform that automates the deployment, scaling, and management of containerized applications.

<p align="center">
    <img src="../images/K8s-basics.png" alt="K8s basics"/>
</p>

**Key Components:**
- **Pods**: Smallest deployable units that contain one or more containers
- **Deployments**: Manage the deployment and scaling of pods
- **Services**: Enable network access to pod sets
- **ConfigMaps/Secrets**: Store configuration data and sensitive information

### Argo-CD concetops and termonologies

Argo CD is a declarative GitOps continuous delivery tool for Kubernetes.

<p align="center">
    <img src="../images/gitops termonologies.png" alt="gitops termonologies"/>
</p>

### CI-CD with GitOps

GitOps uses Git as the single source of truth for infrastructure and application definitions.

<p align="center">
    <img src="../images/cicd with GitOps.png" alt="cicd with GitOps"/>
</p>

**Workflow:**
1. Developers push code changes to application repository
2. CI pipeline builds, tests, and pushes Docker images
3. Image tag/version updated in Kubernetes manifest repository
4. Argo CD detects changes and automatically syncs the cluster state

### Deploy Argo CD
```bash
# Install Argo CD in your Kubernetes cluster
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access Argo CD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Login using admin credentials
argocd login localhost:8080
```

For more information, visit the [Argo CD documentation](https://argo-cd.readthedocs.io/).

### Gitops workflow setup

####  Creating Applications in Argo CD

Before you begin:
- Ensure your Kubernetes manifests are stored in a Git repository
- Have access to your Argo CD instance

**Steps to Create an Application:**

1. **Prepare Your Repository**
   - Organize your Kubernetes manifests in a dedicated directory (e.g., `./kubernetes`)
   - Commit and push your manifests to your Git repository

2. **Access the Argo CD UI**
   - Navigate to your Argo CD instance URL
   - Log in with your credentials

3. **Create a New Application**
   - Click the **+ NEW APP** button in the upper left corner
   - Configure your application with the following settings:

   | Section | Field | Description | Example |
   |---------|-------|-------------|---------|
   | **General** | Name | Unique name for your application | `react-app` |
   | | Project | Project to organize applications | `default` |
   | | Sync Policy | How changes are applied | `Automatic` or `Manual` |
   | | Sync Options | Additional sync behaviors | ✓ `AUTO-CREATE NAMESPACE` |
   | **Source** | Repository URL | Git repository containing manifests | `https://github.com/your-org/your-repo.git` |
   | | Revision | Git branch or tag | `main` or `HEAD` |
   | | Path | Directory containing manifests | `./kubernetes` |
   | **Destination** | Cluster URL | Target Kubernetes cluster | `https://kubernetes.default.svc` |
   | | Namespace | Target namespace | `react-app` |

4. **Review and Create**
   - Verify all settings are correct
   - Click **CREATE** to finish

5. **Sync Your Application**
   - If using manual sync policy, click the **SYNC** button to deploy
   - Argo CD will apply your Kubernetes manifests to the cluster

#### Verifying Deployment

After creating your application:
1. Monitor the sync status on the Applications dashboard
2. Green status indicators show successful synchronization
3. Explore the application topology to view all deployed resources

For troubleshooting sync issues, check the Argo CD logs:
```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server
```
#### Stages 

Make sure that Trigger on Pull Request is checked in the Git Repo.

**Update kubernetes Docker Image Tag:**
```groovy
stage('K8s Update Image Tag'){
    when{
        branch 'PR*'
    }
    environment {
        REPO_ACCESS_TOKEN = credentails('repo-access-token')
    }
    steps{
        sh 'git clone -b main <git-repo-url>'
        dir('<repo-name>/kubernetes'){
            sh '''
                #### Replace Docker Tag ####
                git checkout main
                git checkout -b feature-$BUILD_ID
                sed -i "s#frontend.#frontend/solr-system:$GIT_COMMIT#g" deployment.yml
                cat deployment.yml

                #### Commit and Push to feature branch ####
                git config --global user.email "<user-email>"
                git remote set-url origin http://$REPO_ACCESS_TOKEN@<repo-url-without-http>
                git add .
                git commit -am "Update docker image"
                git push -u origin feature-$BUILD_ID
            '''
        }
        post {
            always{
                script{
                    if (fileExists('<repo-name>')){
                        sh 'rm -rf <repo-name>'
                    }
                }
            }
        }
    }
}
```
**Raise PR to kubernetes Repo:**
```groovy
stage('k8s - Raise PR'){
    when{
        branch 'PR*'
    }
    environment {
        REPO_ACCESS_TOKEN = credentails('repo-access-token')
    }
    steps{
        sh """
            curl -X POST \
                '<repo-url>'
                -H 'accept: application/json' \
                -H 'Authorization: token ${REPO_ACCESS_TOKEN}'
                -H 'Content-Type: application/json'
                -d '{
                    "assignee": "<git-username>",
                    "assignees": [
                        "<git-username>"
                    ],
                    "base":"main",
                    "body": "Updated image in deployment manifest",
                    "head": "feature-${BUILD_ID}",
                    "title": "Updated Docker Image"
                }'
        """
    }
}
```
The endpoint in *k8s - raise PR* stage may differ from Git Host Repo to another, so make sure to check the documentation.

**DAST:**

The DAST checks for attaquest like Cross site scripting, SQL Injection..., for that there is commercial and open-source projects, we will use the Open source **Zed Attack Proxy (ZAP)**, this type of vunerability testing it requires the application to be runnig. 

ZAP can be used in various ways, in this guide we will use the Docker approach, For the Docker approach there are multiple kind of scan,
<p align="center">
    <img src="../images/zap-docker.png" alt="zap docker"/>
</p>

 For application that have frontend and backend it is recommended to use the **Full Scan**, but in this guide we will use the **API Scan**, this kind uses the *openapi(swagger), soap, or Graphql specification*

 ```groovy

 stage('APP Deployed?'){
    when{
        branch 'PR*'
    }
    steps{
        timeout(time: 1, unit: 'DAYS') {
            input message: 'Is the PR Merged and ArgoCD Synced?' ok: 'Yes! PR is Merged an ArgoCD Application is Synced'
        }
    }
 }

 stage('DAST - OWASP ZAP'){
    when{
        branch 'PR*'
    }
    steps{
        sh """
            chmod 777 ${pwd}
            docker run -v ${pwd}:/zap/wrk/:rw ghcr.io/zaproxy/zaproxy zap-api-scan.py \
            -t <backend-url>/api-docs/ \
            -f openapi \
            -r zap_report.html \
            -w zap_report.md \
            -j zap_json_report.json \
            -x zap_xml_report.xml
        """
    }
 }
 ```
 **Note**: the */api-docs/* its a jsob documentation not the UI documentation
