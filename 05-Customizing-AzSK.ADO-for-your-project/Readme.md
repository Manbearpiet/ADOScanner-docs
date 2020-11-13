### [Customizing AzSK.ADO for your project](Readme.md#customizing-azskado-for-your-project-1)
- [Overview](Readme.md#Overview)
- [Setting up org policy](Readme.md#setting-up-org-policy)  
- [Consuming custom org policy](Readme.md#consuming-custom-org-policy)  
- [Modifying and customizing org policy](Readme.md#modifying-and-customizing-org-policy)  
- [Advanced usage of org policy (extending AzSK.ADO)](Readme.md#advanced-usage-of-org-policy-extending-azskado) 
- [Frequently Asked Questions](Readme.md#frequently-asked-questions) 

# Customizing AzSK.ADO for your project

## Overview

#### When and why should I setup org policy

When you run any scan command from AzSK.ADO, it relies on JSON-based policy files to determine various parameters that effect the behavior of the command it is about to run. These policy files are downloaded 'on the fly' from a policy server. When you run the public version of the scanner, the offline policy files present in the module are accessed. Thus, whenever you run a scan from a vanilla installation, AzSK.ADO accesses the offline file present in the module to get the policy configuration and runs the scan using it. 

The JSON inside the policy files dictate the behavior of the security scan. 
This includes things such as:
 - Which set of controls to evaluate?
 - What control set to use as a baseline?
 - What settings/values to use for individual controls? 
 - What messages to display for recommendations? Etc.


Note that the policy files needed for security scans are downloaded into each PS session for **all** AzSK.ADO scenarios. That is, apart from manually-run scans from your desktop, this same behavior happens if you include the 'ADO Security Scanner' extension task in your CICD pipeline. 

 While the out-of-box files in the module may be good for limited use, in many contexts you may want to "customize" the behavior of the security scans for your environment. You may want to do things such as: (a) enable/disable 
some controls, (b) change control settings to better match specific security policies within your project, (c) change various messages, (d) add additional filter criteria for certain regulatory requirements that teams in your project can leverage, etc. When faced with such a need, you need a way to create and manage 
a dedicated policy endpoint customized to the needs of your environment. The organization policy setup feature helps you do that in an automated fashion. 

In this document, we will look at how to setup an org-specific policy endpoint, how to make changes to and manage the policy files and how to accomplish various common org-specific policy/behavior customizations 
for the scanner.

>**Note:** We will be treating **PROJECT** as a boundary to customize scanner behavior. Any customizations made will apply strictly only to the project (and its components) where the org-policy endpoint resides. We will be interchangeably using the terms 'org' and 'project'.  

#### How does AzSK.ADO use online policy?

Let us look at how policy files are leveraged in a little more detail. 

When you install AzSK.ADO, it downloads the latest AzSK.ADO module from the PS Gallery. Along with this module there is an *offline* set of policy files that go in a sub-folder under the %userprofile%\documents\WindowsPowerShell\Modules\AzSK.ADO\<version> folder. It also places (or updates) an AzSKSettings.JSON file in your %LocalAppData%\Microsoft\AzSK.ADO folder that contains the policy endpoint (or policy server) URL that is used by all local commands. 

Whenever any command is run, AzSK.ADO uses the policy server URL to access the policy endpoint. It first downloads a 'metadata' file that contains information about what other files are available on the policy server. After 
that, whenever AzSK.ADO needs a specific policy file to actually perform a scan, it loads the local copy of the policy file into memory and 'overlays' any settings *if* the corresponding file was also found on the 
server-side. 

It then accesses the policy to download a 'metadata' file that helps it determine the actual policy files list that is present on the server. Thereafter, the scan runs by overlaying the settings obtained from the server with 
the ones that are available in the local installation module folder. This means that if there hasn't been anything overridden for a specific feature (e.g., Project), then it won't find a policy file for that listed in the server
 metadata file and the local policy file for that feature will get used. 

## Setting up org policy

#### What happens during org policy setup?

At a high level, the org policy setup support for AzSK.ADO does the following:
 - Sets up a repository to hold various policy artifacts in the project you want to use for hosting your policy endpoint. (This should be a secure, limited-access repo to be used only for managing your project's AzSK.ADO policy.)
 - Uploads the minimum set of policy files required to bootstrap your policy server.

#### Steps to setup org policy setup 

1. Create a Git repository in your project by importing this [repo](https://github.com/azsk/ADOScanner_Policy.git). [Project -> Repos -> Import repository -> Select 'Git' as repository type -> Enter 'https://github.com/azsk/ADOScanner_Policy.git' as clone URL -> Enter 'ADOScannerPolicy' as name].

It will import a very basic 'customized' policy involving below files uploaded to the policy repository.

##### Basic files setup during policy setup 
 
| File | Description  
| ---- | ---- | 
| AzSK.Pre.json | This file contains a setting that controls/defines the AzSK.ADO version that is 'in effect' in a project. A project can use this file to specify the specific version of AzSK.ADO that will get used in SDL/CICD scenarios at the project level.<br/> <br/>  **Note:** Whenever a new AzSK.ADO version is released, the org policy owner should update the AzSK.ADO version in this file with the latest released version after performing any compatibility tests in a test setup.<br/> You can get notified of new releases by following the AzSK.ADO module in PowerShell Gallery or release notes section [here](https://azsk.azurewebsites.net/ReleaseNotes/LatestReleaseNotes.html).   
| AzSK.json | Includes org-specific message, installation command etc.
| ServerConfigMetadata.json | Index file with list of policy files.  

## Consuming custom org policy

Running scan with custom org policy is supported from both avenues of AzSK.ADO viz. local scan (SDL) and ADO security scanner extension task (CICD). Follow the steps below for the same:

### 1. Running scan in local machine with custom org policy

 To run scan with custom org policy from any machine, run the below command

```PowerShell
#Run scan cmdlet and validate if it is running with org policy
Get-AzSKADOSecurityStatus -OrganizationName "<Organization name>" -ProjectNames "<Project name where the org policy is configured>"

#Using 'PolicyProject' parameter
Get-AzSKADOSecurityStatus -OrganizationName "<Organization name>" -PolicyProject "<Name of the project hosting organization policy with which the scan should run.>"

```
> **Note**: Using PolicyProject parameter you can specify the name of the project hosting organization policy with which the scan should run.

### 2. Using ADO security scanner extension with custom org policy

To set up CICD when using custom org policy, add 'ADO Security Scanner' extension in ADO build pipeline by following the steps [here](Readme.md#setting-up-continuous-assurance---step-by-step).

## Modifying and customizing org policy 

#### Getting Started

The typical workflow for all policy changes will remain same and will involve the following basic steps:

 1) Make modifications to the existing files (or add additional policy files as required)
 2) Update *ServerConfigMetadata.json* to include files to be overlayed while running scan commands.
 3) Test in a fresh PS console that the policy change is in effect. (Policy changes do not require re-installation of AzSK.ADO)


Because policy on the server works using the 'overlay' approach, **the corresponding file on the server needs to have only those specific changes that are required (plus some identifying elements in some cases).**

Lastly, note that while making modifications, you should **never** edit the files that came with the AzSK.ADO installation folder %userprofile%\documents\WindowsPowerShell\Modules\AzSK.ADO). 
You should create copies of the files you wish to edit, place them in our org-policy repo and make requisite modifications there.
	
### Basic scenarios for org policy customization


In this section let us look at typical scenarios in which you would want to customize the org policy and ways to accomplish them. 

> Note: To edit policy JSON files, use a friendly JSON editor such as Visual Studio Code. It will save you lot of
> debugging time by telling you when objects are not well-formed (extra commas, missing curly-braces, etc.)! This
> is key because in a lot of policy customization tasks, you will be taking existing JSON objects and removing
> large parts of them (to only keep the things you want to modify).


##### a) Changing the default `'Running AzSK.ADO cmdlet...'` message
Whenever any user in your project runs scan command targetting resources within the project (project/build/release/service connection/agent pool), 
they should see such a message

    Running AzSK.ADO cmdlet...

This message resides in the AzSK.json policy file on the server and AzSK.ADO *always* displays the text from the server version of this file.

You may want to change this message to something more detailed. (Or even use this as a mechanism to notify all users
within the project about something related to AzSK.ADO that they need to attend to immediately.) 
In this example let us just make a change to this message. We will add the project name in the message.

###### Steps:

 i) Open the AzSK.json in the *master* branch of your org-policy repo.
     
 ii) Edit the value for "Policy Message" field by adding the project name as under:
   ```
   "PolicyMessage" : "Running AzSK.ADO cmdlet using <Project name> policy"
   ```
 iii) Commit the file to the *master* branch of the repo. 
 
 > **Note:** 1. Unless explicitly mentioned, we will be referring to the *master* branch of the org-policy repo.  
 > 2. It is recommended to commit updates using a pull request workflow. 

###### Testing:

The updated policy is now on the policy server. You can ask another person to test this by running the below scan cmdlet in a **fresh** PS console.

```PowerShell
#Run scan cmdlet and validate if it is running with org policy
Get-AzSKADOSecurityStatus -OrganizationName "<Organization name>" -ProjectNames "<Project name where the org policy is configured>"
```
 When the command starts, it will show an updated message as in the 
image below:

![Org-Policy - changed message](../Images/09_ADO_Org_Policy1.png) 

This change will be effective across your project immediately. Anyone running AzSK.ADO commands (in fresh PS sessions) should see the new message. 

##### b) Changing a control setting for specific controls 
Let us now change some numeric setting for a control. A typical setting you may want to tweak is the maximum number of
days when a build pipeline was last queued before it is marked inactive.  It is verified in one of the build security controls. (The default value is 180 days.)

This setting resides in a file called ControlSettings.json. Because the first-time org policy setup does not customize anything from this, we will first need to copy this file from the local AzSK.ADO installation.

The local version of this file should be in the following folder:
```PowerShell
    %userprofile%\Documents\WindowsPowerShell\Modules\AzSK.ADO\<version>\Framework\Configurations\SVT
```

   ![Local AzSK.AzureDevOps Policies](../Images/09_ADO_Org_Policy2.png) 
 
Note that the 'Configurations' folder in the above picture holds all policy files (for all features) of AzSK.ADO. We 
will make copies of files we need to change from here and place the changed versions in the org-policy repo. 
Again, you should **never** edit any file directly in the local installation policy folder of AzSK.ADO. 
Rather, **always** copy the file and edit it.

###### Steps:

 i) Copy the ControlSettings.json from the AzSK.ADO installation to your org-policy repo.
 
 ii) Remove everything except the "BuildHistoryPeriodInDays" line while keeping the JSON object hierarchy/structure intact.

  ![Edit Number of build history period in days](../Images/09_ADO_Org_Policy3.png) 

 iii) Commit the file.

 iv) Add an entry for *ControlSettings.json* in *ServerConfigMetadata.json* (in the repo) as shown below.

 ![Update control settings in ServerConfigMetadata](../Images/09_ADO_Org_Policy4.png) 
 
###### Testing: 

Anyone in your project can now start a fresh PS console and the result of the evaluation whether a build pipeline is inactive in 
the build security scan (Get-AzSKADOBuildSecurityStatus) should reflect that the new setting is in 
effect. (E.g., if you change the period to 90 days and if the pipeline was inactive from past 120 days, then the result for control (ADO_Build_SI_Review_Inactive_Build) will change from 'Passed' to 'Failed'.)

##### c) Customizing specific controls for a service 

In this example, we will make a slightly more involved change in the context of a specific SVT (Project). 

Imagine that you want to turn off the evaluation of some control altogether (regardless of whether people use the `-UseBaselineControls` parameter or not).
Also, for another control, you want people to use a recommendation which leverages an internal tool the security team
in your org has developed. Let us do this for the ADO.Project.json file. Specifically, we will:
1. Turn off the evaluation of `ADO_Project_AuthZ_Min_RBAC_Access` altogether.
2. Modify severity of `ADO_Project_AuthZ_Set_Visibility_Private` to `Critical` for our project (it is `High` by default).
3. Change the recommendation for people in our project to follow if they need to address an issue with the `ADO_Project_AuthZ_Review_Group_Members` control.
4. Disable capability to attest the control `ADO_Project_AuthZ_Limit_Job_Scope_To_Current_Project` by adding 'ValidAttestationStates' object.

###### Steps: 
 
 i) Copy the ADO.Project.json from the AzSK,.ADO installation to your org-policy repo

 ii) Remove everything except the ControlID, Id and specific property we want to modify as mentioned above. 

 iii) Make changes to the properties of the respective controls so that the final JSON looks like the below. 

```JSON
{
  "Controls": [
   {
      "ControlID": "ADO_Project_AuthZ_Set_Visibility_Private",
      "Id": "Project110",
      "ControlSeverity": "Critical"
   },
   {
      "ControlID": "ADO_Project_AuthZ_Min_RBAC_Access",
      "Id": "Project120",
      "Enabled": false
   },
   {
      "ControlID": "ADO_Project_AuthZ_Review_Group_Members",
      "Id": "Project130",
      "Recommendation": "**Note**: Use our Contoso-MyProject-ReviewGroups.ps1 tool for this!"
   },
   {
      "ControlID": "ADO_Project_AuthZ_Limit_Job_Scope_To_Current_Project",
      "Id": "Project160",
      "ValidAttestationStates" : ["None"]
   }
  ]
}
```
> **Note:** The 'Id' field is used for identifying the control for policy merging. We are keeping the 'ControlId'
> field only because of the readability.

 iii) Commit the file

 iv) Add an entry for *ADO.Project.json* in *ServerConfigMetadata.json* (in the repo) as shown below.

  ![Update project SVT in ServerConfigMetadata](../Images/09_ADO_Org_Policy5.png) 
 
 
###### Testing: 
Someone in your project can test this change using the `Get-AzSKADOProjectSecurityStatus` command on the project for which we have configured the org policy. If run with the `-UseBaselineControls` switch, you will see that
the control *ADO_Project_AuthZ_Set_Visibility_Private* shows as `Critical` in the output CSV and the recommendation for control *ADO_Project_AuthZ_Review_Group_Members* has changed to
the custom (internal tool) recommendation you wanted people in your project to follow. 

Likewise, even after you run the scan without the `-UseBaselineControls` parameter, you will see that the control *ADO_Project_AuthZ_Min_RBAC_Access* is not evaluated and does not
appear in the resulting CSV file. 





##### d) Customizing Severity labels 
Ability to customize naming of severity levels of controls (e.g., instead of High/Medium, etc. one can now have Important/Moderate, etc.) with the changes reflecting in all avenues (manual scan results/CSV, dashboards, etc.)

###### Steps: 

 i) Edit the ControlSettings.json file to add a 'ControlSeverity' object as per below:
 
```JSON
{
   "ControlSeverity": {
    "Critical": "Critical",
    "High": "Important",
    "Medium": "Moderate",
    "Low": "Low"
  }
}
```
 ii) Commit the file.
 
 iii) Confirm that an entry for ControlSettings.json is already there in the ServerConfigMetadata.json file. (Else see step-iv in (b) above.)


 ###### Testing: 

Someone in your project can test this change using the `Get-AzSKADOSecurityStatus`. You will see that
the controls severity shows as `Important` instead of `High` and `Moderate` instead of `Medium` in the output CSV.

##### e) Modifying a custom control 'baseline' for your project
A powerful capability of AzSK.ADO is the ability for a project to define a baseline control set on the policy server
that can be leveraged by all individuals in the project in both local scan and CICD scenarios via the *-UseBaselineControls* parameter
during scan commands. 

By default, when someone runs the scan with *-UseBaselineControls* parameter, it leverages the set of
controls listed as baseline in the ControlSettings.json file present in the default offline file of module. 

To modify baseline controls for your project, you will need to update the ControlSettings.json
file as per the steps below-
 
(We assume that you have tried the inactive build period steps in (b) above and edited the ControlSettings.json 
file is already present in your org policy repo.)

 i) Edit the ControlSettings.json file to add a 'BaselineControls' object as per below:
 
```JSON
{
    "Build": {
        "BuildHistoryPeriodInDays": 90
    },
    "BaselineControls": {
        "ResourceTypeControlIdMappingList": [
            {
                "ResourceType": "Project",
                "ControlIds": [
                    "ADO_Project_AuthZ_Set_Visibility_Private",
                    "ADO_Project_SI_Limit_Variables_Settable_At_Queue_Time"
                ]
            },
            {
                "ResourceType": "Build",
                "ControlIds": [
                    "ADO_Build_SI_Review_Inactive_Build"
                ]
            }
        ]
    }
}
```

> Notice how, apart from the couple of extra elements at the end, the baseline set is pretty much a list of 'ResourceType'
and 'ControlIds' for that resource...making it fairly easy to customize/tweak your own project baseline. 
> Here the name and casing of the resource type name must match that of the policy JSON file for the corresponding resource's JSON file in the SVT folder and the control ids must match those included in the JSON file. 

> Note: Here we have used a very simple baseline with just a couple of resource types and a very small control set.

 ii) Commit the ControlSettings.json file.
 
 iii) Confirm that an entry for ControlSettings.json is already there in the ServerConfigMetadata.json file. (Else see step-iv in (b) above.)

> Note: This will include new controls that you added in the baseline set on the server side plus the original baseline set (present in the offline file in module). This happens as *ControlSetting.json* file has been **overlayed** on the server side. 

In order to not include the original baseline in the scan, you need to **override** the file on server side. Refer the below steps to override baseline set on server side.

**Steps:**

i) Copy local version of ControlSettings.json file to org policy repo.

Source location: "%userprofile%\Documents\WindowsPowerShell\Modules\AzSK.ADO\<version>\Framework\Configurations\SVT"

ii) Update all the required configurations (baseline set, preview baseline set, build configurations, etc.) in ControlSettings.json

iii) Commit the file.

iv) Add entry for configuration in index file(ServerConfigMetadata.json) with OverrideOffline property as shown here 

![Override ADO Configurations](../Images/09_ADO_Org_Policy6.png)


###### Testing:

To test that the baseline controls set is in effect, anyone in your project can start a fresh PS console and run the project and build security scan cmdlets with the `-UseBaselineControls` parameter.

> **Note:** Similar to baseline control, you can also define preview baseline set with the help of similar property "PreviewBaselineControls" in ControlSettings.json. This preview set gets scanned using parameter `-UsePreviewBaselineControls` with scan commands.
	
#### Troubleshooting common issues 
Here are a few common things that may cause glitches and you should be careful about:

- Make sure you use exact case for file names for various policy files (and the names must match case-and-all
with the entries in the ServerConfigMetadata.json file)
- Make sure that no special/BOM characters get introduced into the policy file text. 

## Advanced usage of org policy (extending AzSK.ADO)

### Customizing the SVTs

It is powerful capability of AzSK.ADO to enable a project to customize the Security Verification Tests (SVT) behaviour. You will be able to achieve the following scenarios.

   - Update/extend existing control by augmenting logic
   - Add new control for existing feature SVT.

   ### Know more about SVTs:


All our SVTs inherit from a base class called SVTBase which will take care of all the required plumbing from the control evaluation code. Every SVT will have a corresponding feature json file under the configurations folder. For example, ADO.Project.ps1 (in the core folder) has a corresponding ADO.Project.json file under configurations folder. These SVTs json have a bunch of configuration parameters, that can be controlled by a policy owner, for instance, you can change the recommendation, modify the description of the control suiting your project, change the severity, etc.

Below is the typical schema for each control inside the feature json
  ```
{
    "ControlID": "ADO_Project_AuthZ_Limit_Job_Scope_To_Current_Project",   //Human friendly control Id. The format used is ADO_<FeatureName>_<Category>_<ControlName>
    "Description": "Scope of access of all pipelines should be restricted to current project.",  //Description for the control, which is rendered in all the reports it generates (CSV, AI telemetry, emails etc.).
    "Id": "Project160",   //This is internal ID and should be unique. Since the ControlID can be modified, this internal ID ensures that we have a unique link to all the control results evaluation.
    "ControlSeverity": "Medium", //Represents the severity of the Control. 
    "Automated": "Yes",   //Indicates whether the given control is Manual/Automated.
    "MethodName": "CheckJobAuthnScope",  // Represents the Control method that is responsible to evaluate this control. It should be present inside the feature SVT associated with this control.
    "Recommendation": "Go to Project Settings --> Pipelines --> Settings --> Enable 'Limit job authorization scope to current project.'.",	  //Recommendation typically provides the precise instructions on how to fix this control.
    "Tags": [
        "SDL",
        "TCP",
        "Automated",
        "AuthZ"
    ], // You can decorate your control with different set of tags, that can be used as filters in scan commands.
    "Enabled": true ,  //Defines whether the control is enabled or not.
    "Rationale": "This ensures pipeline execution happens using a token scoped to the current project abiding with principle of least privilege." //Provides the intent of this control.
}
 ```  
    
After schema of the control json, let us look at the corresponding feature SVT PS1.

```PowerShell
Set-StrictMode -Version Latest
class SubscriptionCore: SVTBase
{
	[PSObject] $PipelineSettingsObj = $null

    Project([string] $subscriptionId, [SVTResource] $svtResource): Base($subscriptionId,$svtResource) 
    {
        $this.GetPipelineSettingsObj()
    }
	.
	.
	.
	hidden [ControlResult] CheckJobAuthnScope([ControlResult] $controlResult)
	{

		#Step 1: This is where the code logic is placed
		#Step 2: ControlResult input to this function, which needs to be updated with the verification Result (Passed/Failed/Verify/Manual/Error) based on the control logic
		Messages that you add to ControlResult variable will be displayed in the detailed log automatically.
		#Step 3: You can directly access the properties from ControlSettings.json. Any property that you add to controlsettings.json will be accessible from your SVT
		
		if($this.PipelineSettingsObj)
       {
            
            if($this.PipelineSettingsObj.enforceJobAuthScope.enabled -eq $true )
            {
                $controlResult.AddMessage([VerificationResult]::Passed, "Scope of access of all pipelines is restricted to current project. It is set as '$($this.PipelineSettingsObj.enforceJobAuthScope.orgEnabled)' at organization scope.");
            }
            else{
                $controlResult.AddMessage([VerificationResult]::Failed, "Scope of access of all pipelines is set to project collection. It is set as '$($this.PipelineSettingsObj.enforceJobAuthScope.orgEnabled)' at organization scope.");
            }       
       }
       else{
            $controlResult.AddMessage([VerificationResult]::Manual, "Pipeline settings object could not be fetched due to insufficient permissions at project scope.");
        }       
        return $controlResult
	}
	.
	.
	.
}
```
	
### Steps to extend the control SVT:
##### A. Extending a GSS SVT
1. 	 Copy the SVT ps1 script that you want to extend and rename the file by replacing "ADO_<Feature>.ps1" with "<Feature>.ext.ps1".
	For example, if you want to extend ADO.Project.ps1, copy the file and rename it to Project.ext.ps1.
  
2. 	 You need to rename the class, inherit from the core feature class, and then update the constructor to reflect the new name as shown below:
    
   > e.g. class Project: ADOSVTBase => ProjectExt : Project
	
   ```PowerShell
	Set-StrictMode -Version Latest
	class ProjectExt : Project
	{
	  ProjectExt([string] $subscriptionId, [SVTResource] $svtResource): Base($subscriptionId, $svtResource) 
	  {

	  }
	}
   ```
   All the other functions from the class file should be removed.
  
3. 	 If you are modifying the logic for a specific control, then retain the required function; or if you are adding a new control, copy any control function from the base class to the extension class reference.
	> Note: For a given control in json, the corresponding PowerShell function is provided as value under MethodName property. You can search for that method under the PS script. For eg. In the below case let us assume you want to add a new control that fails if project visibility is set to private. 
  
  ```PowerShell
	Set-StrictMode -Version Latest
	class ProjectExt: Project
	{
		ProjectExt([string] $subscriptionId, [SVTResource] $svtResource): Base($subscriptionId, $svtResource) 
    {

    }
		hidden [ControlResult] CheckPrivateProjects([ControlResult] $controlResult)
		{
			#This is internal
			$apiURL = $this.ResourceContext.ResourceId;
      $responseObj = [WebRequestHelper]::InvokeGetWebRequest($apiURL);
			
			if([Helpers]::CheckMember($responseObj,"visibility"))
        {
          #Project is visibility is set to private.
            if($responseObj.visibility -eq "Private")
            {
                $controlResult.AddMessage([VerificationResult]::Failed,
                                                "Project visibility is set to $($responseObj.visibility)."); 

            }
            else {
                $controlResult.AddMessage([VerificationResult]::Passed,
                                                "Project visibility is set to $($responseObj.visibility).");
            }
        }
        $controlResult.SetStateData("Project visibility is set to ", $responseObj.visibility);
        return $controlResult;
		}
	}
  ```  
  
  4. 	Now you need to prepare the json for the above new control. You can get started by copying the default base json, rename it to ADO_<Feature>.ext.json. In this case you need to rename it as ADO_Project.ext.json. Remove all the other controls except for one and update it with new control details. See additional instructions as '//comments' on each line in the example JSON below. Note: Remove the comments from JSON if you happen to use the below as-is.
	
  > IMPT: Do *not* tag 'Ext' to the 'FeatureName' here. Make sure you have updated the MethodName to the new method name. 
  > Note: Remove the comments in the below JSON before saving the file
  
``` 
	{
    "FeatureName":  "Project",
    "Reference":  "aka.ms/azsktcp/project",
    "IsMaintenanceMode":  false,
	   "Controls": [
            {
                "ControlID": "ADO_Project_AuthZ_Dont_Set_Visibility_Private",  //define the new control id
                "Description": "Ensure that project visibility is not set to private", //Description for your control
                "Id": "Project180", //Ensure that all the internal ids are appended with 4 digit integer code
                "ControlSeverity": "Medium", //Control the severity
                "Automated": "Yes",  // Control the automation status
                "MethodName": "CheckPrivateProjects", //Update the method name with the new one provided above
                "Recommendation": "Refer: https://docs.microsoft.com/en-us/azure/devops/organizations/public/make-project-public?view=vsts&tabs=new-nav",
                "Tags": [
                      "SDL",
                      "TCP",
                      "Automated",
                      "AuthZ"
                ],
                "Enabled": true,
                "Rationale": "Data/content in projects that have public visibility can be downloaded by anyone on the internet without authentication. This can lead to a compromise of corporate data."
            }
	   ]
	}
```
5. 	 Now upload these files to your org policy repo and add an entry to ServerConfigMetadata.json as shown below:
``` 
    {
      "Name":  "ADO.Project.ext.json"
    },
    {
      "Name":  "Project.ext.ps1"
    }
```  
  
6. 	 That's it!! You can now scan the new extended control like any other control.
  
```PowerShell
	Get-AzSKADOSecurityStatus -OrganizationName "org_name" -ProjectNames "<Org_Policy_Project_Name>" -ControlIds 'Azure_Subscription_AuthZ_Limit_Admin_Count_Ext'
```

### Steps to override the logic of existing SVT:

1. Add new Feature.ext.ps1/Project.ext.ps1 file with the new function that needs to be executed as per the above documentation.
2. Customize ADO.Feature.json/ADO.Project.json file as per by overriding "MethodName" property value with the new function name that needs to be executed.
3. That's it!! You can now scan the older control with overridden functionality. 

### Steps to add extended control in baseline control list:

1. Add new control to ADO.Feature.json/ADO.Project.json file as per the above documentation.
2. Add the new ControlId in baseline control list.
3. That's it!! The newly added control will be scanned while passing "-UseBaselineControls" switch to GSS/GRS cmdlets.


## Frequently Asked Questions

#### Can I completely override policy. I do not want policy to be run in Overlay method?

Yes. You can completely override policy configuration with the help of index file. 

**Steps:**

i) Copy local version of configuration file to org policy repo. Here we will copy complete ControlSettings.json. 

Source location: "%userprofile%\Documents\WindowsPowerShell\Modules\AzSK.ADO\<version>\Framework\Configurations\SVT"

ii) Update all the required configurations (baseline set, preview baseline set, build configurations, etc.) in ControlSettings.json

iii) Commit the file.

iv) Add entry for configuration in index file(ServerConfigMetadata.json) with OverrideOffline property as shown here 

![Override ADO Configurations](../Images/09_ADO_Org_Policy6.png)

### Control is getting scanned even though it has been removed from my custom org-policy.

If you want only the controls which are present on your custom org-policy to be scanned, set the  OverrideOffline flag to true in the ServerConfigMetadata.json file.
