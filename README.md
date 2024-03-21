# ACAmslearnbicepdemo

This bicep file is copied from MS  learn exercise : https://learn.microsoft.com/en-us/azure/container-apps/blue-green-deployment?pivots=bicep

targetScope = 'resourceGroup'
param location string = resourceGroup().location

@minLength(1)
@maxLength(64)
@description('Name of containerapp')
param appName string

@minLength(1)
@maxLength(64)
@description('Container environment name')
param containerAppsEnvironmentName string

@minLength(1)
@maxLength(64)
@description('CommitId for blue revision')
param blueCommitId string

@maxLength(64)
@description('CommitId for green revision')
param greenCommitId string = ''

@maxLength(64)
@description('CommitId for the latest deployed revision')
param latestCommitId string = ''

@allowed([
  'blue'
  'green'
])
@description('Name of the label that gets 100% of the traffic')
param productionLabel string = 'blue'

var currentCommitId = !empty(latestCommitId) ? latestCommitId : blueCommitId

resource containerAppsEnvironment 'Microsoft.App/managedEnvironments@2022-03-01' existing = {
  name: containerAppsEnvironmentName
}

resource blueGreenDeploymentApp 'Microsoft.App/containerApps@2022-11-01-preview' = {
  name: appName
  location: location
  tags: {
    blueCommitId: blueCommitId
    greenCommitId: greenCommitId
    latestCommitId: currentCommitId
    productionLabel: productionLabel
  }
  properties: {
    environmentId: containerAppsEnvironment.id
    configuration: {
      maxInactiveRevisions: 10 // Remove old inactive revisions
      activeRevisionsMode: 'multiple' // Multiple active revisions mode is required when using traffic weights
      ingress: {
        external: true
        targetPort: 80
        traffic: !empty(blueCommitId) && !empty(greenCommitId) ? [
          {
            revisionName: '${appName}--${blueCommitId}'
            label: 'blue'
            weight: productionLabel == 'blue' ? 100 : 0
          }
          {
            revisionName: '${appName}--${greenCommitId}'
            label: 'green'
            weight: productionLabel == 'green' ? 100 : 0
          }
        ] : [
          {
            revisionName: '${appName}--${blueCommitId}'
            label: 'blue'
            weight: 100
          }
        ]
      }
    }
    template: {
      revisionSuffix: currentCommitId
      containers:[
        {
          image: 'mcr.microsoft.com/k8se/samples/test-app:${currentCommitId}'
          name: appName
          resources: {
            cpu: json('0.5')
            memory: '1.0Gi'
          }
          env: [
            {
              name: 'REVISION_COMMIT_ID'
              value: currentCommitId
            }
          ]
        }
      ]
    }
  }
}

output fqdn string = blueGreenDeploymentApp.properties.configuration.ingress.fqdn
output latestRevisionName string = blueGreenDeploymentApp.properties.latestRevisionName


#Deploy an blue environment
export APP_NAME=<APP_NAME>
export APP_ENVIRONMENT_NAME=<APP_ENVIRONMENT_NAME>
export RESOURCE_GROUP=<RESOURCE_GROUP>

# A commitId that is assumed to belong to the app code currently in production
export BLUE_COMMIT_ID=fb699ef
# A commitId that is assumed to belong to the new version of the code to be deployed
export GREEN_COMMIT_ID=c6f1515

# create a new app with a blue revision
az deployment group create --name createapp-$BLUE_COMMIT_ID --resource-group $RESOURCE_GROUP --template-file main.bicep --parameters appName=$APP_NAME blueCommitId=$BLUE_COMMIT_ID containerAppsEnvironmentName=$APP_ENVIRONMENT_NAME --query properties.outputs.fqdn

#deploy a new version of the app to green revision
az deployment group create --name deploy-to-green-$GREEN_COMMIT_ID --resource-group $RESOURCE_GROUP --template-file main.bicep --parameters appName=$APP_NAME blueCommitId=$BLUE_COMMIT_ID greenCommitId=$GREEN_COMMIT_ID latestCommitId=$GREEN_COMMIT_ID productionLabel=blue containerAppsEnvironmentName=$APP_ENVIRONMENT_NAME --query properties.outputs.fqdn

#Send full traffic to green deployment
# make green the prod revision
az deployment group create --name make-green-prod-$GREEN_COMMIT_ID --resource-group $RESOURCE_GROUP --template-file main.bicep --parameters appName=$APP_NAME blueCommitId=$BLUE_COMMIT_ID greenCommitId=$GREEN_COMMIT_ID latestCommitId=$GREEN_COMMIT_ID productionLabel=green containerAppsEnvironmentName=$APP_ENVIRONMENT_NAME --query properties.outputs.fqdn
