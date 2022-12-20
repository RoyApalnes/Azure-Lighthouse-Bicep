Want to deploy everything with templates from DevOps, but each task in a pipeline only connects to one subscription?

The solution is configuring Azure Lighthouse with a service principal (app registration), and configure Service Connections in DevOps with the service principal. This makes sure the service principal receives a token for each subscription added to Azure Lighthouse, when connecting to one of them, which was necessary in order to achieve VNET Peering across tenants.

Here is the guide to how you setup Azure Lighthouse, register App Registration and create the Service Connection in Azure DevOps.

Pre-req: Decide which tenant and subscription shall be the hub with access to all other subscriptions. Important as you might add users with Portal Access to all subscriptions, and not only a DevOps Pipeline.

Before you read on, this an old post being re-posted due to error in wordpress. So there have been changes you might stumble upon. For instance

Agenda:

1. Create and configure an Application Registration to gain access using a secret.
2. Create a template- and parameter-file for connecting other subscriptions to Lighthouse in the hub subscription.
3. Register the AppReg in all other tenants.
4. Create a custom role for the AppReg (or use builtin roles).
5. Deploy the Lighthouse Template- and Parameter-file.
6. Create the Service Connections in Azure DevOps.
7. Test DevOps with Lighthouse

Let's GO!

Step 1 Create and configure an Application Registration to gain access using a secret.

Create a new AppReg from Azure Active Directory.

Requires minimum Application Administrator Role.

Choose multitenant configuration, because we need to register this AppReg in all tenants we wanne deploy our network peering to.

1.2 Create a Secret and copy the secret to notepad, as you will only see it once after creation.

1.3 Achieve the Service Principal Name ID, and copy the Id to notepad for use later.

Get-AzADServicePrincipal -DisplayName "DevOps Service Connection with Lighthouse"

Next up is; Create a template- and parameter-file for connecting other subscriptions to Lighthouse in the hub subscription.

Step 2. Create a template- and parameter-file for connecting other subscriptions to Lighthouse in the hub subscription.

Copy this sample template and parameter code:

Lighthouse Template sample: .\Lighthouse-Template.json

Lighthouse Parameter sample: .\Lighthouse-Parameter.json

2.1 Modify the Parameter-file to your hub, app registration and roles:

Note: Microsoft have the Az-module today.

manageByTenantID = TenantId

PrincipalID = Service Principal ID, copied to notepad. Can also be User Object ID, if your also giving named users access cross subscription.

roleDefintionId = Builtin or custom Role Object ID. (b24988ac-6180-42a0-ab88-20f7382dd24c = Subscription Contributor).

Step 3 Register the AppReg in all other tenants.

3.1 Modify this URL to fit your the spoke tenantID and the AppRegApplicationID:

https://login.microsoftonline.com/tenantID/oauth2/authorize?client_id=AppRegApplicationID&response_type=code&redirect_uri=https://microsoft.com

Sample: https://login.microsoftonline.com/15ae15c5-ffd5-1be5-a1cb-cb15aa15bc15/oauth2/authorize?client_id=16ae16c6-ffd6-1be6-a1cb-cb16aa16bc16&response_type=code&redirect_uri=https://microsoft.com

3.2 Open a browser tab where you can login as Application Administrator (minimum) to the spoke tenants, and go to that URL. Follow the steps to confirm adding the AppReg and do so for each tenant hosting subscriptions that shall be onboarded to Lighthouse.

Step 4 Create a custom role for the AppReg (or use builtin roles).

4.1 Modify this InputFile to create the role and access you as owner of the spoke subscription would like to give the Service Principal and Users in Lighthouse.

{
    "Name": "Network Resource Group Contributor",
    "IsCustom": true,
    "Description": "Contributor Role for the Network Resource Group.",
    "Actions": [
      "Microsoft.Resources/subscriptions/resourceGroups/read",
      "Microsoft.Resources/subscriptions/resourcegroups/resources/read",
      "Microsoft.Resources/subscriptions/resourcegroups/deployments/operations/read",
      "Microsoft.Resources/subscriptions/resourcegroups/deployments/operationstatuses/read",
      "Microsoft.Resources/subscriptions/resourcegroups/deployments/read",
      "Microsoft.Resources/subscriptions/resourcegroups/deployments/write",
      "Microsoft.Resources/subscriptions/resourceGroups/write",
      "Microsoft.Resources/subscriptions/resourceGroups/delete"
    ],
    "NotActions": [],
    "DataActions": [],
    "NotDataActions": [],
    "AssignableScopes": [
    "/subscriptions/16ae16c6-ffd6-1be6-a1cb-cb16aa16bc16/resourceGroups/CustomRoy"
    ]
  }

This would give the Service Principal and Users access to manage resource and deployments in this specified Resource Group within that subscription.

In my case I was only going to create the hub-spok network, so to reduce risk, one can only deploy to this specific Resource Group, that should host the networking resources.

4.2 Run the New-AzRoleDefintion cmdlet to create a role based on this InputFile

Step 5 Deploy the Lighthouse Template- and Parameter-file.

#Yes, I am using AzureRM rather than the Az PS-module, because communcating to multiple customers/owners its efficient to have all use the Azure Shell and it doesnt have the Az-module yet. Makes sure it works for everyone, no matter which PS-module they may have installed on their clients.

5.1 Upload the previous modified template and parameter-files to where your able to run an Azure Deployment from.

Can be anywhere, but I prefer having all owners use the same and recommend them to use the Azure Cloud Shell.

5.2 Change to the directory your files where uploaded to:

5.3 From a PowerShell window with the module AzureRM, like the Azure Cloud Shell, run an ARM Deployment.

New-AzDeployment -Name Lighthouse -TemplateFile ./Lighthouse-Template.json -TemplateParameterFile ./Lighthouse-Parameter.json -Location WestEurope

We will get the AzModule soon enough, so dont worry about the warning. This is all controlled by Microsoft within the Azure Cloud Shell, hence we cannot do it ourselves.

Step 6 Create the Service Connections in Azure DevOps.

6.1 Collect all values necessary for each Subscription you onboarded to Lighthouse, as I am deploying a virtual network to each of them.

During previous posts, we have copied them to notepad, so heres a short recap of what a DevOps Service Connection to Azure Resource Manager requires:

SubscriptionName, SubscriptionID and TenantID: Using PowerShell (Post 1/6) or navigate to Azure Active Directory Properties.
ServicePrincipalClientID: Using PowerShell (Post 1/6) or navigate to the Application Registration Overview to copy the ApplicationID/ClientID.
Secret: Copied to notepad when created i Post 1/6, but you can go ahead and create a new one if you lost it.

6.2 Navigate to your DevOps Project and Add a Service Connections under Project Settings.
6.2.1 Create a New Connection and choose Azure Resource Manager.
6.2.2 Choose to use full version of the service connection dialog.

Step 7 Test your DevOps setup using ARM template with Lighthouse

DevOps ARM template deployment testing the benefit you get from onboarding tenants/subscriptions to Lighthouse, or solution to my customers initial challenge, peering VNETs cross tenants using ARM/IaC.

Initial setup of the hub and the first spoke, requires these four templates/parameter-files:

Hub-Template and Parameter.json
Spoke-Template and Parameter.json
Hub-Peering-Template and Parameter.json
Spoke-Peering-Template and Parameter.json

Adding additional Spokes, requires these four templates/parameter-files:

Spoke-Template and Parameter.json
Hub-Peering-Template and Parameter.json
Spoke-Peering-Template and Parameter.json