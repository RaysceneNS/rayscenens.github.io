---
title: "Tagging Git History from Azure DevOps Release pipelines"
tags: [Git, Azure, DevOps]
---

In order to generate a tag in git that marked the commit of the source code that was actually released to my Production Environment. I create this small powershell command that is able to mark the commit using the Azure rest API. The benefit of this approach is that I don't need to clone the repository in my Release pipeline and no external components or dependencies are required. 

The $TagName and $TagMessage variables are configured so that the format of the tag will be the release name followed by the environment name i.e. 'Release-11-DEV'
But you can change the format of the tags or messages to suite any situation.

This script requires access to the OAuth token from within your pipeline, this is used to supply the Invoke-RestMethod with the Bearer token authentication header.

To enable this you must go to your Release pipeline in Azure Devops and select the Agent that you will run on ![screen0](/assets/images/2021/3/29/screen0.png) The enable the option 'Allow scripts to access the OAuth token' checkbox of the agent property panel for the agent that will run this task.
![screen1](/assets/images/2021/3/29/screen1.png)

After that is done simply add a new PowerShell Script task ![screen2](/assets/images/2021/3/29/screen2.png) to your existing release pipeline ![screen3](/assets/images/2021/3/29/screen3.png) and paste the following script into it:

```PowerShell
$TagName = "$env:RELEASE_RELEASENAME-$env:SYSTEM_STAGEDISPLAYNAME"
$TagName.replace(" ","-")
$TagMessage = "$env:RELEASE_RELEASENAME to $env:SYSTEM_STAGEDISPLAYNAME"

$body = [ordered]@{
    name = $TagName
    taggedObject = @{
        objectId = $env:BUILD_SOURCEVERSION
    }
    message = $TagMessage
   } |ConvertTo-Json -Depth 2
$url = "$env:SYSTEM_TEAMFOUNDATIONCOLLECTIONURI$env:SYSTEM_TEAMPROJECTID/_apis/git/repositories/$env:BUILD_REPOSITORY_ID/annotatedtags?api-version=6.0-preview.1"

Try {
    Invoke-RestMethod -Method Post -Uri $url -ContentType "application/json" -Headers @{ Authorization = "Bearer $env:SYSTEM_ACCESSTOKEN" } -Body $body
}
Catch {
    if ($_.Exception.Response.StatusCode.value__ -eq 409) {
        Write-Warning "Tag '$TagName' already exists."
    }
    else {
        throw $_
    }
}
```
