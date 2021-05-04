---
layout: post
title:  "Why is Microsoft not recommending a pipeline variable?"
date:   2020-03-03 18:00:00 +0000
categories: ci/cd azure devops microsoft
---

[Microsoft documentation](https://docs.microsoft.com) is pretty awesome these days. Many parts are open-source so edits can be by submitted thorough a pull request. Unfortunately this is not available for [Microsoft Learn preparation for AZ400 exam (Azure DevOps)](https://docs.microsoft.com/en-us/learn/certifications/exams/az-400?wt.mc_id=learningredirect_certs-web-wwl). Hence this blog post. 


When demonstrating how a variable can be used to take conditional actions in a pipeline, Microsoft doc recommends to use a Variable Group in Library to store whether database schema has changed or not.
<https://docs.microsoft.com/en-gb/learn/modules/manage-database-changes-in-azure-pipelines/index>

This variable has to be saved using the Azure Devops REST API:

~~~powershell
Install-Module VSTeam -Scope CurrentUser -Force
Set-VSTeamAccount –Account $(Acct) -PersonalAccessToken $(PAT)
$methodParameters = @{
  ProjectName = "$(System.TeamProject)"
  Name = "Release"}
$vg = Get-VSTeamVariableGroup  @methodParameters 
$vars = @{}
$vg.variables | Get-Member -MemberType *Property | %{$vars.($_.Name) = $vg.variables.($_.Name)}
$vars.Remove("schemaChanged")
$methodParameters = @{
  id = $vg.id
  ProjectName = "$(System.TeamProject)"
  Name = "Release"
  Description = ""
  Type = "Vsts"
  Variables = $vars}
Update-VSTeamVariableGroup @methodParameters
~~~


## Suggestion 
**Would it not make more sense to use a pipeline variable than a variable in a variable group ?**

Some of the advantages of the pipeline variable I can see are: 
* A Variable Group in Library is akin to a global variable. It is visible to the entire Azure Devops project. A pipeline variable is limited in scope to this pipeline. 
* Reading and writing to a Variable Group requires use of the REST API and a PAT token. Interacting with a pipeline variable needs none of these. 

Updating a pipeline variable from powershell task is documented in [this Stackoverflow question](https://stackoverflow.com/questions/58286114/azure-devops-set-build-numbeTher-variable-in-a-build-task)


We could simply set the variable with the following in deployment step of `DBAVerificationScript` job.
~~~powershell
Write-Host "##vso[task.setvariable variable=schemaChanged;]$containsWord"
~~~

The conditon in stage `DBAVerificationApply` could remain unchanged:
~~~yaml
condition: and(succeeded('DBAVerificationScript'), eq(variables['schemaChanged'], True))
~~~

## Edit: Response from an author of Microsoft AZ-400 exam training material
> There is use case for both variable groups and variables with in the pipeline. It's best to choose a variable group if you want to abstract the value and make it usable across multiple pipelines and also if this value needs to be secured then the variable groups support that while a status value in pipeline variable is plain text.