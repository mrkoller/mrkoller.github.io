---
layout: post
title: 'Azure App Registration Owners via Graph PowerShell'
date: 2024-05-31
tags: [azure, powershell]
permalink: azure-appreg-owner-powershell
image: /assets/img/app-owner-diagram.png
---

Delegation of permissions is often essential for applications management. You might be required to add owners to apps and this can easily be done. However, caution should be exercised when adding owners to Entra application registrations because owners can add other users, manage users for the app, among [other things](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/overview-assign-app-owners). 

Once you have determined if your situation is appropriate you might choose to add owners to apps. If you have to do this for more than a few instances, using PowerShell or some other automated method is easiest. I recently had the need to add multiple owners to multiple apps (adding groups is not supported), so I wrote this script in case this comes up again.

The script will:
* Read a csv file of the owner UPN(s)
* Run a search for the apps to to be edited (alternatively get a list of apps from a csv)
* Get the app Ids of the applications
* Assign owner(s) to that application

You’ll need:
* The Application Administrator Entra RBAC role or the consented scopes defined below.
* A csv list of user(s) to be added as owner of the application.
* A search to identify the apps you want to add the owners or a csv of the apps
* Microsoft.Graph.Applications Graph PowerShell module
* Microsoft.Graph.Users Graph PowerShell module

For my case all the apps I wanted to target were named through a standard naming convention, so it was easiest to identify them by a quick search. If your apps do not have a standardized naming convention, I have included the code to use a csv list.  Just comment/uncomment out the appropriate lines and it should work all the same.

Once you meet all the requirements for the files and search criteria, this should get all the users added.

```powershell
<#
    .SUMMARY
	    Add potentially multiple owners to multiple application registrations in Azure

    .DETAILS
        This uses Graph PowerShell commands to get the required information from the user you provide in a csv file and adds
        those users as Owners to all the applications you provide the search criteria for. Note there are security implications
        to adding people as Owner of applications so please understand the consequences of this first.
        https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/overview-assign-app-owners

        You'll need to edit the search query to apply to your use case. If your use case does not fit into search
        neatly, comment that line out and uncomment the needed line(s) and supply the path to your .csv.
        *Note* In my testing exporting application registrations from the Azure portal as a .csv will not give you all the
        possible applications, so search is superior. 

    .REQUIREMENTS
        Microsoft.Graph.Applications Module
        Microsoft.Graph.Users Module
        At least Application Administrator RBAC role or specific scope permissions granted
        A csv file of SamAccountNames or the UPN aside from the domain section
        Editing the application search criteria or commenting it out and using a csv of app Ids instead.

    .AUTHOR
        Matt Koller

    .VERSION
        1.0

    .DATE
        04.17.2024
#>

# Import required modules
Import-Module Microsoft.Graph.Applications
Import-Module Microsoft.Graph.Users

# Connect to Graph (Need App Admin for this script)
Connect-MgGraph -Scope “User.Read.All”,”Group.ReadWrite.All”

# Assuming the CSVs have columns 'Username' for the user file and 'Id' for the applications file
# You could do this if you could not accurately search for the applications in scope, otherwise leave commented out and update search below
# $applicationsCsv = "path_to_applications.csv"
$appOwnerUsers = "C:\Temp\AppOwnerUsers.csv"

# Get the list of application Ids you want to add the users to using search
$applications = Get-MgApplication -ConsistencyLevel eventual -Search '"DisplayName:dynamics365-appreg"'

$applicationIds = $applications | Select-Object -Property Id
# $applicationIds | Export-Csv -Path "appRegsIds.csv" -NoTypeInformation

# Reading the user and application data from CSV files
try {
    $users = Import-Csv -Path $appOwnerUsers -ErrorAction Stop
    #$applicationIds = Import-Csv -Path $applicationsCsv -ErrorAction Stop
}
catch {
    Write-Error "Failed to read CSV files: $_"
    exit
}

# Loop through each user
foreach ($user in $users) {
    # Get the users objectId in Azure
    $userId = (Get-MgUser -ConsistencyLevel eventual -Filter "startsWith(UserPrincipalName, '$($user.Username)')").Id
    $NewOwner = @{
        "@odata.id"= "https://graph.microsoft.com/v1.0/directoryObjects/{$userId}"
        }

    # Loop through each application
    foreach ($application in $applicationIds) {
        try {
            New-MgApplicationOwnerByRef -ApplicationId $application.Id -BodyParameter $NewOwner -ErrorAction Stop
            Write-Output "Successfully added user $($user.Username) to application $($application.Id)."
        }
        catch {
            Write-Error "Failed to add user $($user.Username) to application $($application.Id): $_"
        }
    }
}
Disconnect-MgGraph
```
 