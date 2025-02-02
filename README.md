# .net-core-cli
trigger:
- main

variables:
  - name: workingDirectory
    value: terraform-example-deploy

stages:
- stage: validate
  displayName: Validation
  condition: eq(variables['Build.Reason'], 'PullRequest')
  jobs:
    - job:
      displayName: Validate Terraform
      pool:
        vmImage: ubuntu-latest
      steps:
      - task: TerraformInstaller@0
        displayName: Install Terraform
        inputs:
          terraformVersion: 'latest'
      - pwsh: terraform fmt -check
        displayName: Terraform Format Check
        workingDirectory: $(workingDirectory)
      - pwsh: terraform init -backend=false
        displayName: Terraform Init
        workingDirectory: $(workingDirectory)
      - pwsh: terraform validate
        displayName: Terraform Validate
        workingDirectory: $(workingDirectory)
      
- stage: deploy_to_dev
  displayName: Deploy to Dev
  condition: ne(variables['Build.Reason'], 'PullRequest')
  variables:
    - group: dev
    - name: serviceConnection
      value: service_connection_dev
  jobs:
    - deployment: deploy
      displayName: Deploy with Terraform
      pool:
        name: agent-pool-dev
      environment: dev
      strategy:
        runOnce:
          deploy:
            steps:
            - checkout: self
              displayName: Checkout Terraform Module
            - task: TerraformInstaller@0
              displayName: Install Terraform
              inputs:
                terraformVersion: 'latest'
            - task: TerraformTaskV4@4
              displayName: Terraform Init
              inputs:
                provider: 'azurerm'
                command: 'init'
                workingDirectory: '$(workingDirectory)'
                backendServiceArm: '${{ variables.serviceConnection }}'
                backendAzureRmResourceGroupName: '$(BACKEND_AZURE_RESOURCE_GROUP_NAME)'
                backendAzureRmStorageAccountName: '$(BACKEND_AZURE_STORAGE_ACCOUNT_NAME)'
                backendAzureRmContainerName: '$(BACKEND_AZURE_STORAGE_ACCOUNT_CONTAINER_NAME)'
                backendAzureRmKey: 'terraform.tfstate'
              env:
                ARM_USE_AZUREAD: true # This environment variable tells the backend to use AzureAD auth rather than trying a create a key. It means we can limit the permissions applied to the storage account and container to least priviledge: https://developer.hashicorp.com/terraform/language/settings/backends/azurerm#use_azuread_auth
            - task: TerraformTaskV4@4
              displayName: Terraform Apply
              inputs:
                provider: 'azurerm'
                command: 'apply'
                workingDirectory: '$(workingDirectory)'
                commandOptions: '-auto-approve -var="resource_group_name=$(AZURE_RESOURCE_GROUP_NAME)"'
                environmentServiceNameAzureRM: '${{ variables.serviceConnection }}'
              env:
                ARM_USE_AZUREAD: true
      
- stage: deploy_to_test
  displayName: Deploy to Test
  condition: and(not(or(failed(), canceled())), ne(variables['Build.Reason'], 'PullRequest'))
  dependsOn: deploy_to_dev
  variables:
    - group: test
    - name: serviceConnection
      value: service_connection_test
  jobs:
    - deployment: deploy
      displayName: Deploy with Terraform
      pool:
        name: agent-pool-test
      environment: test
      strategy:
        runOnce:
          deploy:
            steps:
            - checkout: self
              displayName: Checkout Terraform Module
            - task: TerraformInstaller@0
              displayName: Install Terraform
              inputs:
                terraformVersion: 'latest'
            - task: TerraformTaskV4@4
              displayName: Terraform Init
              inputs:
                provider: 'azurerm'
                command: 'init'
                workingDirectory: '$(workingDirectory)'
                backendServiceArm: '${{ variables.serviceConnection }}'
                backendAzureRmResourceGroupName: '$(BACKEND_AZURE_RESOURCE_GROUP_NAME)'
                backendAzureRmStorageAccountName: '$(BACKEND_AZURE_STORAGE_ACCOUNT_NAME)'
                backendAzureRmContainerName: '$(BACKEND_AZURE_STORAGE_ACCOUNT_CONTAINER_NAME)'
                backendAzureRmKey: 'terraform.tfstate'
              env:
                ARM_USE_AZUREAD: true
            - task: TerraformTaskV4@4
              displayName: Terraform Apply
              inputs:
                provider: 'azurerm'
                command: 'apply'
                workingDirectory: '$(workingDirectory)'
                commandOptions: '-auto-approve -var="resource_group_name=$(AZURE_RESOURCE_GROUP_NAME)"'
                environmentServiceNameAzureRM: '${{ variables.serviceConnection }}'
              env:
                ARM_USE_AZUREAD: true

- stage: deploy_to_prod
  displayName: Deploy to Prod
  condition: and(not(or(failed(), canceled())), ne(variables['Build.Reason'], 'PullRequest'))
  dependsOn: deploy_to_test
  variables:
    - group: prod
    - name: serviceConnection
      value: service_connection_prod
  jobs:
    - deployment: deploy
      displayName: Deploy with Terraform
      pool:
        name: agent-pool-prod
      environment: prod
      strategy:
        runOnce:
          deploy:
            steps:
            - checkout: self
              displayName: Checkout Terraform Module
            - task: TerraformInstaller@0
              displayName: Install Terraform
              inputs:
                terraformVersion: 'latest'
            - task: TerraformTaskV4@4
              displayName: Terraform Init
              inputs:
                provider: 'azurerm'
                command: 'init'
                workingDirectory: '$(workingDirectory)'
                backendServiceArm: '${{ variables.serviceConnection }}'
                backendAzureRmResourceGroupName: '$(BACKEND_AZURE_RESOURCE_GROUP_NAME)'
                backendAzureRmStorageAccountName: '$(BACKEND_AZURE_STORAGE_ACCOUNT_NAME)'
                backendAzureRmContainerName: '$(BACKEND_AZURE_STORAGE_ACCOUNT_CONTAINER_NAME)'
                backendAzureRmKey: 'terraform.tfstate'
              env:
                ARM_USE_AZUREAD: true
            - task: TerraformTaskV4@4
              displayName: Terraform Apply
              inputs:
                provider: 'azurerm'
                command: 'apply'
                workingDirectory: '$(workingDirectory)'
                commandOptions: '-auto-approve -var="resource_group_name=$(AZURE_RESOURCE_GROUP_NAME)"'
                environmentServiceNameAzureRM: '${{ variables.serviceConnection }}'
              env:
                ARM_USE_AZUREAD: true
