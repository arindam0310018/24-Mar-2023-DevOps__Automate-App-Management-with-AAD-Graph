# Automate App Management with AAD Graph and DevOps

Greetings to my fellow Technology Advocates and Specialists.

In this Session, I will demonstrate __How to Automate App Management with AAD Graph and DevOps.__

| THIS IS HOW IT LOOKS:- |
| --------- |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/btqxz1z27p594sslsodc.jpg) |

| __AUTOMATION OBJECTIVES:-__ |
| --------- |

| __#__ | __TOPICS__ |
| --------- | --------- |
|  1. | Validate if the Resource Group and the Key Vault residing in it exists. |
|  2. | Validate if the App Registration already exists. If No, App Registration will be created. |
|  3. | Secret will be generated and stored in Key Vault. |
|  4. | Set Redirect URI and Enable ID Token. |
|  5. | Set Token Configuration - Optional Claims. |
|  6. | Set Token Configuration - Groups Claim. |
|  7. | Set Microsoft Graph API Permissions. |
|  8. | Create App Roles. |
|  9. | Set App Owners. |

| __IMPORTANT NOTE:-__ |
| --------- |
The YAML Pipeline is tested on __WINDOWS BUILD AGENT__ Only!!!
 
| __REQUIREMENTS:-__ |
| --------- |
1. Azure Subscription.
2. Azure DevOps Organisation and Project.
3. Service Principal with Required RBAC ( __Contributor__) applied on Subscription or Resource Group(s). 
4. Azure Resource Manager Service Connection in Azure DevOps.
5. Azure Tenant by type "Azure Active Directory (AAD)" with one of the Licenses: a.) Azure AD Premium P2, OR b.) Enterprise Mobility + Security (EMS) E5 license.
6. "__Cloud Application Administrator__" is required to create and configure App Registration.
7. "__Global Administrator__" PIM role is required to Grant Admin Consent to Microsoft Graph API Rights.
8. A test Azure Active Directory (AAD) user to add as an owner of the App. 

| __HOW DOES MY CODE PLACEHOLDER LOOKS LIKE:-__ |
| --------- |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nletx1e658b4c7a10rrm.png) |

| PIPELINE CODE SNIPPET:- | 
| --------- |

| AZURE DEVOPS YAML PIPELINE (azure-pipelines-app-registration-setup-config-v1.1.yml):- |
| --------- |

```
trigger:
  none

######################
#DECLARE PARAMETERS:-
######################
parameters:
- name: SubscriptionID
  displayName: Subscription ID details follows below:-
  type: string
  default: 210e66cb-55cf-424e-8daa-6cad804ab604
  values:
  - 210e66cb-55cf-424e-8daa-6cad804ab604

- name: RGName
  displayName: Please Provide the Resource Group Name:-
  type: object
  default: 

- name: KVName
  displayName: Please Provide the Keyvault Name:-
  type: object
  default: 

- name: SPIName
  displayName: Please Provide the Service Principal Name:-
  type: object
  default:

- name: RedirectURI
  displayName: Please Provide Redirect URL:-
  type: object
  default: https://contoso.azurewebsites.net/.auth/login/aad/callback

- name: Username
  displayName: Please Provide the Username:-
  type: object
  default: U1@mitra008.onmicrosoft.com

######################
#DECLARE VARIABLES:-
######################
variables:
  ServiceConnection: amcloud-cicd-service-connection
  BuildAgent: windows-latest
  Claims: "optionalclaims.json"
  MSGraphAPI: 00000003-0000-0000-c000-000000000000
  MSGraphUsrRead: e1fe6dd8-ba31-4d61-89e7-88639da4683d
  MSGraphEmail: 64a6cdd6-aab1-4aaf-94b8-3cc8405e90d0
  MSGraphProfile: 14dad69e-099b-42c9-810b-d002981feec1
  MSGraphUsrReadAll: a154be20-db9c-4678-8ab7-66f6cc099a59
  MSGraphGrpReadAll: 5f8c59db-677d-491f-a6b8-5f174b11ec1d
  AppRoles: "approles.json"


#########################
# Declare Build Agents:-
#########################
pool:
  vmImage: $(BuildAgent)

###################
# Declare Stages:-
###################

stages:

- stage: App_Registration_Setup_Configure 
  jobs:
  - job: App_Registration_Setup_Configure 
    displayName: App Registration Setup and Configure
    steps:
    - task: AzureCLI@2
      displayName: Validate and Create App
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          az --version
          az account set --subscription ${{ parameters.SubscriptionID }}
          az account show
          $i = az ad sp list --display-name ${{ parameters.SPIName }} --query [].appDisplayName -o tsv
          if ($i -ne "${{ parameters.SPIName }}") {
            $j = az group exists -n ${{ parameters.RGName }}
                if ($j -eq "true") {
                  $k = az keyvault list --resource-group ${{ parameters.RGName }} --query [].name -o tsv		
                      if ($k -eq "${{ parameters.KVName }}") {
                        $spipasswd = az ad sp create-for-rbac -n ${{ parameters.SPIName }} --query "password" -o tsv
                        az keyvault secret set --name ${{ parameters.SPIName }} --vault-name ${{ parameters.KVName }} --value $spipasswd
                        echo "##################################################################"
                        echo "Service Principal ${{ parameters.SPIName }} created successfully and the Secret Stored inside Key Vault ${{ parameters.KVName }} in the Resource Group ${{ parameters.RGName }}!!!"
                        echo "##################################################################"
                        }				
                      else {
                      echo "##################################################################"
                      echo "Key Vault ${{ parameters.KVName }} DOES NOT EXISTS in Resource Group ${{ parameters.RGName }}!!!"
                      echo "##################################################################"
                      exit 1
                          }
                }
                else {
                echo "##################################################################"
                echo "Resource Group ${{ parameters.RGName }} DOES NOT EXISTS!!!"
                echo "##################################################################"
                exit 1
                    }
          }
          else {
          echo "##################################################################"
          echo "Service Principal ${{ parameters.SPIName }} EXISTS and hence Cannot Proceed with Deployment!!!"
          echo "##################################################################"
          exit 1
              }
    - task: AzureCLI@2
      displayName: Set Redirect URI & ID Token
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          $appid = az ad app list --display-name ${{ parameters.SPIName }} --query [].appId -o tsv
          az ad app update --id $appid --web-redirect-uris ${{ parameters.RedirectURI }} --enable-id-token-issuance true
          echo "##################################################################"
          echo "Redirect URL is set and ID Token has been enabled successfully!!!"
          echo "##################################################################"
    - task: AzureCLI@2
      displayName: Token Config - Optional Claims
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          $appid = az ad app list --display-name ${{ parameters.SPIName }} --query [].appId -o tsv
          az ad app update --id $appid --optional-claims $(System.DefaultWorkingDirectory)/App-Registration-Setup-Config/$(Claims)
          echo "##################################################################"
          echo "Token Configuration - Optional Claims configured successfully!!!"
          echo "##################################################################"
    - task: AzureCLI@2
      displayName: Token Config - Groups Claims
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          $appid = az ad app list --display-name ${{ parameters.SPIName }} --query [].appId -o tsv
          az ad app update --id $appid --set groupMembershipClaims=SecurityGroup
          echo "##################################################################"
          echo "Token Configuration - Groups Claim has been enabled successfully!!!"
          echo "##################################################################"
    - task: AzureCLI@2
      displayName: MS Graph API Permissions
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          $appid = az ad app list --display-name ${{ parameters.SPIName }} --query [].appId -o tsv
          az ad app permission add --id $appid --api $(MSGraphAPI) --api-permissions $(MSGraphUsrRead)=Scope
          az ad app permission add --id $appid --api $(MSGraphAPI) --api-permissions $(MSGraphEmail)=Scope
          az ad app permission add --id $appid --api $(MSGraphAPI) --api-permissions $(MSGraphProfile)=Scope
          az ad app permission add --id $appid --api $(MSGraphAPI) --api-permissions $(MSGraphUsrReadAll)=Scope
          az ad app permission add --id $appid --api $(MSGraphAPI) --api-permissions $(MSGraphGrpReadAll)=Scope
          echo "##################################################################"
          echo "MS Graph API Permissions has been configured successfully!!!"
          echo "##################################################################"
    - task: AzureCLI@2
      displayName: App Roles
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          $appid = az ad app list --display-name ${{ parameters.SPIName }} --query [].appId -o tsv
          az ad app update --id $appid --app-roles $(System.DefaultWorkingDirectory)/App-Registration-Setup-Config/$(AppRoles)
          echo "##################################################################"
          echo "App Roles Created Successfully!!!"
          echo "##################################################################"
    - task: AzureCLI@2
      displayName: Set App Owner
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          $appid = az ad app list --display-name ${{ parameters.SPIName }} --query [].appId -o tsv
          $uid = az ad user show --id ${{ parameters.Username }} --query "id" -o tsv
          az ad app owner add --id $appid --owner-object-id $uid
          echo "##################################################################"
          echo "Owner ${{ parameters.Username }} added successfully!!!"
          echo "##################################################################"
```
Now, let me explain each part of YAML Pipeline for better understanding.

| PART #1:- |
| --------- |

| BELOW FOLLOWS PIPELINE RUNTIME VARIABLES CODE SNIPPET:- |
| --------- |

```
######################
#DECLARE PARAMETERS:-
######################
parameters:
- name: SubscriptionID
  displayName: Subscription ID details follows below:-
  type: string
  default: 210e66cb-55cf-424e-8daa-6cad804ab604
  values:
  - 210e66cb-55cf-424e-8daa-6cad804ab604

- name: RGName
  displayName: Please Provide the Resource Group Name:-
  type: object
  default: 

- name: KVName
  displayName: Please Provide the Keyvault Name:-
  type: object
  default: 

- name: SPIName
  displayName: Please Provide the Service Principal Name:-
  type: object
  default:

- name: RedirectURI
  displayName: Please Provide Redirect URL:-
  type: object
  default: https://contoso.azurewebsites.net/.auth/login/aad/callback

- name: Username
  displayName: Please Provide the Username:-
  type: object
  default: U1@mitra008.onmicrosoft.com
```

| PART #2:- |
| --------- |

| BELOW FOLLOWS PIPELINE VARIABLES CODE SNIPPET:- |
| --------- |

```
######################
#DECLARE VARIABLES:-
######################
variables:
  ServiceConnection: amcloud-cicd-service-connection
  BuildAgent: windows-latest
  Claims: "optionalclaims.json"
  MSGraphAPI: 00000003-0000-0000-c000-000000000000
  MSGraphUsrRead: e1fe6dd8-ba31-4d61-89e7-88639da4683d
  MSGraphEmail: 64a6cdd6-aab1-4aaf-94b8-3cc8405e90d0
  MSGraphProfile: 14dad69e-099b-42c9-810b-d002981feec1
  MSGraphUsrReadAll: a154be20-db9c-4678-8ab7-66f6cc099a59
  MSGraphGrpReadAll: 5f8c59db-677d-491f-a6b8-5f174b11ec1d
  AppRoles: "approles.json"
```

| __NOTE:-__ |
| --------- |
| Please change the values of the variables accordingly. |
| The entire YAML pipeline is build using Runtime Parameters and Variables. No Values are Hardcoded. |
| For all Permissions and Ids, please refer the MS Documentation [link](https://learn.microsoft.com/en-us/graph/permissions-reference#all-permissions-and-ids) |

| PART #3:- |
| --------- |

| This is a Single Stage Pipeline - __App_Registration_Setup_Configure__ (with 7 Pipeline Tasks):- |
| --------- |

| PIPELINE TASK #1:- |
| --------- |
| Validate Resource Group and Key Vault. If either one of the Azure Resource is not Available, Pipeline will fail. | 
| Validate App Registration. If available with same name, Pipeline will fail else App Registration will be created successfully. The generated secret will be stored in the Keyvault. |

```
- stage: App_Registration_Setup_Configure 
  jobs:
  - job: App_Registration_Setup_Configure 
    displayName: App Registration Setup and Configure
    steps:
    - task: AzureCLI@2
      displayName: Validate and Create App
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          az --version
          az account set --subscription ${{ parameters.SubscriptionID }}
          az account show
          $i = az ad sp list --display-name ${{ parameters.SPIName }} --query [].appDisplayName -o tsv
          if ($i -ne "${{ parameters.SPIName }}") {
            $j = az group exists -n ${{ parameters.RGName }}
                if ($j -eq "true") {
                  $k = az keyvault list --resource-group ${{ parameters.RGName }} --query [].name -o tsv		
                      if ($k -eq "${{ parameters.KVName }}") {
                        $spipasswd = az ad sp create-for-rbac -n ${{ parameters.SPIName }} --query "password" -o tsv
                        az keyvault secret set --name ${{ parameters.SPIName }} --vault-name ${{ parameters.KVName }} --value $spipasswd
                        echo "##################################################################"
                        echo "Service Principal ${{ parameters.SPIName }} created successfully and the Secret Stored inside Key Vault ${{ parameters.KVName }} in the Resource Group ${{ parameters.RGName }}!!!"
                        echo "##################################################################"
                        }				
                      else {
                      echo "##################################################################"
                      echo "Key Vault ${{ parameters.KVName }} DOES NOT EXISTS in Resource Group ${{ parameters.RGName }}!!!"
                      echo "##################################################################"
                      exit 1
                          }
                }
                else {
                echo "##################################################################"
                echo "Resource Group ${{ parameters.RGName }} DOES NOT EXISTS!!!"
                echo "##################################################################"
                exit 1
                    }
          }
          else {
          echo "##################################################################"
          echo "Service Principal ${{ parameters.SPIName }} EXISTS and hence Cannot Proceed with Deployment!!!"
          echo "##################################################################"
          exit 1
              }
```

| PIPELINE TASK #2:- |
| --------- |
| Set Redirect URI and Enable ID Token |
| For more details, please refer the MS Documentation [link](https://learn.microsoft.com/en-us/cli/azure/ad/app?view=azure-cli-latest#az-ad-app-update) |

```
- task: AzureCLI@2
      displayName: Set Redirect URI & ID Token
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          $appid = az ad app list --display-name ${{ parameters.SPIName }} --query [].appId -o tsv
          az ad app update --id $appid --web-redirect-uris ${{ parameters.RedirectURI }} --enable-id-token-issuance true
          echo "##################################################################"
          echo "Redirect URL is set and ID Token has been enabled successfully!!!"
          echo "##################################################################"
```

| PIPELINE TASK #3:- |
| --------- |
| Token Configuration: Optional Claims |
| For more details, please refer the MS Documentation [link](https://learn.microsoft.com/en-us/cli/azure/ad/app?view=azure-cli-latest#az-ad-app-update) |

```
- task: AzureCLI@2
      displayName: Token Config - Optional Claims
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          $appid = az ad app list --display-name ${{ parameters.SPIName }} --query [].appId -o tsv
          az ad app update --id $appid --optional-claims $(System.DefaultWorkingDirectory)/App-Registration-Setup-Config/$(Claims)
          echo "##################################################################"
          echo "Token Configuration - Optional Claims configured successfully!!!"
          echo "##################################################################"
```

| PIPELINE TASK #4:- |
| --------- |
| Token Configuration: Groups Claims. |
| For more details, please refer the MS Documentation [link](https://learn.microsoft.com/en-us/cli/azure/ad/app?view=azure-cli-latest#az-ad-app-update) |

```
- task: AzureCLI@2
      displayName: Token Config - Groups Claims
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          $appid = az ad app list --display-name ${{ parameters.SPIName }} --query [].appId -o tsv
          az ad app update --id $appid --set groupMembershipClaims=SecurityGroup
          echo "##################################################################"
          echo "Token Configuration - Groups Claim has been enabled successfully!!!"
          echo "##################################################################"
```

| CONTENTS OF JSON FILE (optionalclaims.json) USED IN PIPELINE TASK 3 and 4:- |
| --------- |

```
{
    "idToken": [
        {
            "name": "email",
            "essential": false
        },
		{
            "name": "family_name",
            "essential": false
        },
		{
            "name": "given_name",
            "essential": false
        },
		{
            "name": "onprem_sid",
            "essential": false
        },
		{
            "name": "sid",
            "essential": false
        },
		{
            "name": "upn",
            "essential": false
        },
		{
            "name": "groups",
            "additionalProperties": ["dns_domain_and_sam_account_name", "emit_as_roles"]
        }
    ],
    "accessToken": [
        {
            "name": "email",
            "essential": false
        },
		{
            "name": "family_name",
            "essential": false
        },
		{
            "name": "given_name",
            "essential": false
        },
		{
            "name": "onprem_sid",
            "essential": false
        },
		{
            "name": "sid",
            "essential": false
        },
		{
            "name": "upn",
            "essential": false
        },
        {
            "name": "groups",
            "additionalProperties": ["dns_domain_and_sam_account_name", "emit_as_roles"]
        }
    ]
    
}
```

| PIPELINE TASK #5:- |
| --------- |
| Set Microsoft Graph API Permissions. |
| For all Permissions and Ids, please refer the MS Documentation [link](https://learn.microsoft.com/en-us/graph/permissions-reference#all-permissions-and-ids) |

```
- task: AzureCLI@2
      displayName: MS Graph API Permissions
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          $appid = az ad app list --display-name ${{ parameters.SPIName }} --query [].appId -o tsv
          az ad app permission add --id $appid --api $(MSGraphAPI) --api-permissions $(MSGraphUsrRead)=Scope
          az ad app permission add --id $appid --api $(MSGraphAPI) --api-permissions $(MSGraphEmail)=Scope
          az ad app permission add --id $appid --api $(MSGraphAPI) --api-permissions $(MSGraphProfile)=Scope
          az ad app permission add --id $appid --api $(MSGraphAPI) --api-permissions $(MSGraphUsrReadAll)=Scope
          az ad app permission add --id $appid --api $(MSGraphAPI) --api-permissions $(MSGraphGrpReadAll)=Scope
          echo "##################################################################"
          echo "MS Graph API Permissions has been configured successfully!!!"
          echo "##################################################################"
```

| PIPELINE TASK #6:- |
| --------- |
| Create App Roles. |
| For more details, please refer the MS Documentation [link](https://learn.microsoft.com/en-us/cli/azure/ad/app?view=azure-cli-latest#az-ad-app-update) |

```
- task: AzureCLI@2
      displayName: App Roles
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          $appid = az ad app list --display-name ${{ parameters.SPIName }} --query [].appId -o tsv
          az ad app update --id $appid --app-roles $(System.DefaultWorkingDirectory)/App-Registration-Setup-Config/$(AppRoles)
          echo "##################################################################"
          echo "App Roles Created Successfully!!!"
          echo "##################################################################"
```

| CONTENTS OF JSON FILE (approles.json) USED IN PIPELINE TASK 6:- |
| --------- |

```
[{
    "allowedMemberTypes": [
      "User"
    ],
    "description": "Readers",
    "displayName": "Readers",
    "isEnabled": "true",
    "value": "Reader"
}]
```

| PIPELINE TASK #7:- |
| --------- |
| Set Owner to App Registration. |
| For more details, please refer the MS Documentation [link](https://learn.microsoft.com/en-us/cli/azure/ad/app/owner?view=azure-cli-latest) |

```
- task: AzureCLI@2
      displayName: Set App Owner
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          $appid = az ad app list --display-name ${{ parameters.SPIName }} --query [].appId -o tsv
          $uid = az ad user show --id ${{ parameters.Username }} --query "id" -o tsv
          az ad app owner add --id $appid --owner-object-id $uid
          echo "##################################################################"
          echo "Owner ${{ parameters.Username }} added successfully!!!"
          echo "##################################################################"
```

__NOW ITS TIME TO TEST!!!__

| TEST CASES:- |
| --------- |
| 1. Pipeline executed successfully. |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xajoqwexpuoft03lzxfl.jpg) |
| 2. App Registration created successfully. |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/204iq9g4d39oewbiuxia.jpg) |
| 3. App Registration Authentication - Redirect URI has been set correctly. ID Token Enabled. |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/007v0sudke20cs0zyus4.jpg) |
| 4. App Registration Secret Generated successfully. |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gmj6g38hwzle54iqftnr.jpg) |
| 5. App Registration Secret stored successfully in Key Vault. |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3bxgdavoiyluq1tdzvj5.jpg) |
| 6. App Registration Token Configuration (Optional Claims and Group Claims) was setup successfully. |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2o07lkccrwj8hooyg1cf.jpg) |
| 7. App Registration API Permissions set correctly. |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qcwdq2wvkyxbpbyxkg5v.jpg) |
| __Note:-__ Global Administrator PIM role is required to Grant Admin Consent to the MS Graph API permissions added. After Pipeline is executed, Cloud Administrator needs to elevate to Global Administrator PIM role to grant admin consent. |
| 8. Below is how it looks after granting admin consent. |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2288ekdk4ppgmv7azju6.jpg) |
| 9. App Registration App Roles created successfully. |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zhqp9ad06rcvke7h3m2p.jpg) |
| 10. App Registration owner added successfully. |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/x91hdknacjasw5qb1fab.jpg) |

__Hope You Enjoyed the Session!!!__

__Stay Safe | Keep Learning | Spread Knowledge__
