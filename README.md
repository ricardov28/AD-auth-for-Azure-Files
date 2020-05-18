## This PowerShell script will do the following:

# Perform a quick pre-req check

# Domain-Join your Azure storage account, creating a computer object in your on-prem AD. Make sure you have permissions to write in on-prem AD.

# Give you Windows NTFS full permissions at the file share level. Will also create an Azure SMB rol with elevated permissions for this user at Azure file share level.

# Mount the Azure file share authenticating with your credentials.

# After it mounts, feel free to start managing Windows NTFS permissions to the mounted file share.

# More info: https://docs.microsoft.com/en-us/azure/storage/files/storage-files-identity-auth-active-directory-enable

# By Ricardo Vazquez



## Sub Id
$SubscriptionId = '<your-subscription-id-here>'
## Resource group where storage account is
$ResourceGroupName = '<resource-group-name-here>'
## Storage account name already existent
$StorageAccountName = '<storage-account-name-here>'
## Azure file share name already existent
$sharename = '<file-share-name-here>'
## On-Prem AD OU already existent. Example#1: OU=STORAGE, Example#2: OU=USA,DC=TEXAS,DC=DALLAS
$OUpath = '<OU-path>'
# Your user will get full permissions in Windows NTFS and Azure SMB
$youremail = '<your@email>'
# Drive letter to mount share. Make sure is not in use. Example P:, Z:, etc
$driveletter = '<desired-drive-letter>'
# Azure storage access key. Azure portal > storage account > Settings > Access Key > Key 1 or Key 2
$storageaccountkey = '<storage-access-key>'
# Path where you downloaded AzFilesHybridPath PowerShell Module. Example C:\myscripts  Download  from https://github.com/Azure-Samples/azure-files-samples/releases
$AzFilesHybridPath = '<download-path>'


Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope CurrentUser

cd $AzFilesHybridPath

Unblock-File .\CopyToPSPath.ps1 

Import-Module -Name AzFilesHybrid

Connect-AzAccount

Select-AzSubscription -SubscriptionId $SubscriptionId 

Debug-AzStorageAccountAuth -StorageAccountName $StorageAccountName -ResourceGroupName $ResourceGroupName -Verbose

Join-AzStorageAccountForAuth `
        -ResourceGroupName $ResourceGroupName `
        -Name $StorageAccountName `
        -DomainAccountType "ComputerAccount" `
        -OrganizationalUnitDistinguishedName "$OUpath"

$FileShareContributorRole = Get-AzRoleDefinition "Storage File Data SMB Share Elevated Contributor" 

$scope = '/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.Storage/storageAccounts/$StorageAccountName/fileServices/default/fileshares/$sharename'

New-AzRoleAssignment -SignInName $youremail -RoleDefinitionName $FileShareContributorRole.Name -Scope /subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.Storage/storageAccounts/$StorageAccountName/fileServices/default/fileshares/$sharename


net use $driveletter \\$StorageAccountName.file.core.windows.net\$sharename /user:"Azure\$StorageAccountName" $storageaccountkey


ICACLS $driveletter\ /grant ($youremail + ':(OI)(CI)F') /T

net use $driveletter /delete /y

net use $driveletter \\$StorageAccountName.file.core.windows.net\$sharename -Persist

