#TestWebsite



# .NET Core Function App to Windows on Azure
# Build a .NET Core function app and deploy it to Azure as a Windows function App.
# Add steps that analyze code, save build artifacts, deploy, and more:
# https://docs.microsoft.com/en-us/azure/devops/pipelines/languages/dotnet-core

trigger:
- main

variables:
  # Azure Resource Manager connection created during pipeline creation
  azureSubscription: '907bfd40-da89-4ed5-9c6d-56fb1ae1ec83'

  # Function app name
  functionAppName: 'MyAzurefuntionApp'

  # Agent VM image name
  vmImageName: 'windows-latest'
  # vmImageName: 'windows-2019'

  # Working Directory
  workingDirectory: '$(System.DefaultWorkingDirectory)/FunctionApp_Isolated_6/FunctionApp_Isolated_6'

stages:
- stage: Build
  displayName: Build stage

  jobs:
  - job: Build
    displayName: Build
    pool:
      vmImage: $(vmImageName)

    steps:
    - task: DotNetCoreCLI@2
      displayName: Build
      inputs:
        command: 'build'
        projects: |
          $(workingDirectory)/*.csproj
        arguments: --output $(System.DefaultWorkingDirectory)/publish_output --configuration Release

    - task: ArchiveFiles@2
      displayName: 'Archive files'
      inputs:
        rootFolderOrFile: '$(System.DefaultWorkingDirectory)/publish_output'
        includeRootFolder: false
        archiveType: zip
        archiveFile: $(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip
        replaceExistingArchive: true

    - publish: $(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip
      artifact: drop

- stage: Deploy
  displayName: Deploy stage
  dependsOn: Build
  condition: succeeded()

  jobs:
  - deployment: Deploy
    displayName: Deploy
    environment: 'development'
    pool:
      vmImage: $(vmImageName)

    strategy:
      runOnce:
        deploy:

          steps:
          - task: AzureFunctionApp@1
            displayName: 'Azure functions app deploy'
            inputs:
              azureSubscription: '$(azureSubscription)'
              appType: functionApp
              appName: $(functionAppName)
              package: '$(Pipeline.Workspace)/drop/$(Build.BuildId).zip'
