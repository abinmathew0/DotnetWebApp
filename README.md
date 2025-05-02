# .NET CI/CD Pipeline Project Documentation (Full Guide)

## 📘 Project Overview
This guide provides a full end-to-end walkthrough for building, deploying, and maintaining a .NET web application using Azure DevOps with a complete CI/CD pipeline. It’s crafted for developers and DevOps engineers to:

- Create a .NET web app  
- Use Git branching strategy (feature → develop → main)  
- Set up a self-hosted agent in Azure DevOps  
- Create Azure Web Apps for Dev, Staging, and Prod  
- Configure a multi-stage YAML pipeline with approvals  

---

## 🛠️ Step 1: Create a Simple Web App with .NET

**Install .NET SDK**  
Download from: https://dotnet.microsoft.com/en-us/download

**Create the Project**
```bash
dotnet new webapp -n SimpleWebApp
cd SimpleWebApp
```

**Run the App Locally**
```bash
dotnet run
```
Visit `http://localhost:5000`

---

## 🧪 Step 2: Add Unit Test Project and Contact Page Feature

**Create Unit Test Project**
```bash
cd ..
dotnet new xunit -n SimpleWebApp.Tests
dotnet add SimpleWebApp.Tests/SimpleWebApp.Tests.csproj reference SimpleWebApp/SimpleWebApp.csproj
```

**Create Contact Page Feature Branch**
```bash
git checkout -b feature/contact-page
dotnet new page -n Contact
```

Update layout to include a link to `/Contact`.

**Commit and Push**
```bash
git add .
git commit -m "Add Contact Page"
git push -u origin feature/contact-page
```

---

## 🧑‍💻 Step 3: Setup Git and GitHub Repository

```bash
git init
git remote add origin <your-github-repo-url>
git checkout -b main
git add .
git commit -m "Initial commit"
git push -u origin main

git checkout -b develop
git push -u origin develop
git checkout -b feature/initial
```

---

## 🏗️ Step 4: Set Up Azure Resources

```bash
az group create --name DotNetAppRG --location eastus
az appservice plan create --name AppServicePlan --resource-group DotNetAppRG --sku B1 --is-linux
az webapp create --name Dev-DotNetProject --resource-group DotNetAppRG --plan AppServicePlan --runtime "DOTNET|9.0"
az webapp create --name Staging-DotNetProject --resource-group DotNetAppRG --plan AppServicePlan --runtime "DOTNET|9.0"
az webapp create --name Prod-DotNetProject --resource-group DotNetAppRG --plan AppServicePlan --runtime "DOTNET|9.0"
```

---

## 🤖 Step 5: Set Up a Self-Hosted Agent

**Generate PAT in Azure DevOps**, then run:
```bash
mkdir agent && cd agent
wget https://vstsagentpackage.azureedge.net/agent/3.220.2/vsts-agent-linux-x64-3.220.2.tar.gz
tar zxvf vsts-agent-linux-x64-3.220.2.tar.gz
./config.sh
./svc.sh install
./svc.sh start
```

---

## 🧠 Step 6: Git Branching Strategy

- `main`: Production-ready  
- `develop`: Integration branch  
- `feature/*`: Developer features  
```
feature/* → develop → main
```

---

## 🔧 Step 7: Create Azure DevOps Pipeline

```yaml
trigger:
  branches:
    include:
      - main
      - develop
      - 'feature/*'

variables:
  buildConfiguration: 'Release'
  artifactName: 'drop'

stages:
- stage: Build
  jobs:
  - job: BuildJob
    pool:
      name: Default
    steps:
    - task: UseDotNet@2
      inputs:
        packageType: 'sdk'
        version: '9.0.x'
    - script: dotnet restore SimpleWebApp/SimpleWebApp.csproj
    - script: dotnet build SimpleWebApp/SimpleWebApp.csproj --configuration $(buildConfiguration)
    - script: dotnet test SimpleWebApp/SimpleWebApp.csproj --no-build --configuration $(buildConfiguration)
    - script: dotnet publish SimpleWebApp/SimpleWebApp.csproj --configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)
    - task: PublishPipelineArtifact@1
      inputs:
        targetPath: '$(Build.ArtifactStagingDirectory)'
        artifact: '$(artifactName)'
        publishLocation: 'pipeline'

- stage: Deploy_Dev
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/develop'))
  dependsOn: Build
  jobs:
  - deployment: DevDeploy
    environment: 'Dev'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: DownloadPipelineArtifact@2
            inputs:
              artifact: '$(artifactName)'
              path: '$(Pipeline.Workspace)'
          - task: AzureWebApp@1
            inputs:
              azureSubscription: 'Azure subscription 1'
              appType: 'webApp'
              appName: 'Dev-DotNetProject'
              package: '$(Pipeline.Workspace)/$(artifactName)/**/*.zip'

- stage: Deploy_Staging
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/develop'))
  dependsOn: Deploy_Dev
  jobs:
  - deployment: StagingDeploy
    environment: 'Staging'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: DownloadPipelineArtifact@2
            inputs:
              artifact: '$(artifactName)'
              path: '$(Pipeline.Workspace)'
          - task: AzureWebApp@1
            inputs:
              azureSubscription: 'Azure subscription 1'
              appType: 'webApp'
              appName: 'Staging-DotNetProject'
              package: '$(Pipeline.Workspace)/$(artifactName)/**/*.zip'

- stage: Deploy_Prod
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  dependsOn: Build
  jobs:
  - deployment: ProdDeploy
    environment: 'Prod'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: DownloadPipelineArtifact@2
            inputs:
              artifact: '$(artifactName)'
              path: '$(Pipeline.Workspace)'
          - task: AzureWebApp@1
            inputs:
              azureSubscription: 'Azure subscription 1'
              appType: 'webApp'
              appName: 'Prod-DotNetProject'
              package: '$(Pipeline.Workspace)/$(artifactName)/**/*.zip'
```


## ✅ Conclusion

This repo demonstrates full CI/CD:
- .NET App → Azure DevOps → Azure Web Apps  
- Branch strategy, self-hosted agent, automated deploys  
- Ideal for onboarding & production-ready builds  

Let's build smarter, faster, and safer! 💪
