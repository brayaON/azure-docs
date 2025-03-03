---
title: How to use the BypassDirSyncOverrides feature of an Azure AD tenant
description: Describes how to use bypassdirsyncoverrides tenant feature to restore synchronization of Mobile and OtherMobile attributes from on-premises Active Directory.
services: active-directory
author: billmath
ms.date: 01/26/2023
ms.author: billmath
ms.topic: how-to
ms.service: active-directory
ms.workload: identity
ms.custom: has-azure-ad-ps-ref
ms.subservice: hybrid
---

# How to use the BypassDirSyncOverrides feature of an Azure AD tenant.

This article describes the _BypassDirsyncOverrides_  feature and how to restore synchronization of Mobile and otherMobile attributes from Azure AD to on-premises Active Directory.

Generally, synchronized users cannot be changed from Azure or Microsoft 365 admin portals, neither through PowerShell using AzureAD or MSOnline modules. The exception to this is the Azure AD user’s attributes called _MobilePhone_ and _AlternateMobilePhones_. These attributes are synchronized from on-premises Active Directory attributes mobile and otherMobile, respectively, but end users can update their own phone number in _MobilePhone_ attribute in Azure AD through their profile page. Admins can also update synchronized user’s _MobilePhone_ and _AlternateMobilePhones_ values in Azure AD using MSOnline PowerShell module.  

Giving users and admins the ability to update phone numbers directly in Azure AD enables enterprises to reduce the administrative overhead of managing user’s phone numbers in local Active Directory as these can change more frequently.

The caveat however, is that once a synchronized user's _MobilePhone_ or _AlternateMobilePhones_ number is updated via admin portal or PowerShell, the synchronization API will no longer honor updates to these attributes when they originate from on-premises Active Directory. This is commonly known as a _“DirSyncOverrides”_ feature. Administrators will notice this behavior when updates to Mobile or otherMobile attributes in Active Directory, do not update the correspondent user’s MobilePhone or AlternateMobilePhones in Azure AD accordingly, even though, the object is successfully synchronized through Azure AD Connect's engine.

## Identifying users with different Mobile and otherMobile values

You can export a list of users with different Mobile and otherMobile values between Active Directory and Azure Active Directory using _‘Compare-ADSyncToolsDirSyncOverrides’_ from _ADSyncTools_ PowerShell module. This will allow you to determine the users and respective values that are different between on-premises Active Directory and Azure Active Directory. This is important to know because enabling the _BypassDirSyncOverrides_ feature will overwrite all the different values in Azure Active Directory with the value coming from on-premises Active Directory.

### Using Compare-ADSyncToolsDirSyncOverrides

As a prerequisite you need to be running Azure AD Connect version 2 or later and install the latest ADSyncTools module from PowerShell Gallery with the following command:

```powershell
Install-Module ADSyncTools 
```

To compare all the synchronized user’s Mobile and OtherMobile values, run the following command:

```powershell
Compare-ADSyncToolsDirSyncOverrides -Credential $(Get-Credential) 
```

>[!NOTE]
> The target API used by this feature does not handle authentication user interactions. MFA or conditional policies will block authentication. When prompted to enter credentials, please use a Global Administrator account that doesn't have MFA enabled or any Conditional Access policy applied. As a last resort, please create a temporary Global Administrator user account without MFA or Conditional Access that can be deleted after completing the desired operations using the BypassDirSyncOverridees feature.

This function will export a CSV file with a list of users where Mobile or OtherMobile values in on-premises Active Directory are different than the respective MobilePhone or AlternateMobilePhones in Azure AD.

At this stage you can use this data to reset the values of the on-premises Active Directory _Mobile_ and _otherMobile_ properties to the values that are present in Azure Active Directory. This way you can capture the most updated phone numbers from Azure AD and persist this data in on-premises Active Directory, before enabling _BypassDirSyncOverrides_ feature. To do this, import the data from the resulting CSV file and then use the _'Set-ADSyncToolsDirSyncOverrides'_ from _ADSyncTools_ module to persist the value in on-premises Active Directory.

For example, to import data from the CSV file and extract the values in Azure AD for a given UserPrincipalName, use the following command:

```powershell
$upn = '<UserPrincipalName>' 
$user = Import-Csv 'ADSyncTools-DirSyncOverrides_yyyyMMMdd-HHmmss.csv' | 
where UserPrincipalName -eq $upn | 
select UserPrincipalName,*InAAD  
Set-ADSyncToolsDirSyncOverridesUser -Identity $upn -MobileInAD $user.MobileInAAD
```

## Enabling BypassDirSyncOverrides feature

By default, _BypassDirSyncOverrides_ feature is turned off. Enabling _BypassDirSyncOverrides_ allows your tenant to bypass any changes made in _MobilePhone_ or _AlternateMobilePhones_ by users or admins directly in Azure AD and always honor the values present in on-premises Active Directory _Mobile_ or _OtherMobile_.

If you do not wish to have end users updating their own mobile phone number or there is no requirement to have admins updating mobile or alternative mobile phone numbers using PowerShell, you should leave the feature _BypassDirsyncOverrides_ enabled on the tenant.  

With this feature turned on, even if an end user or admin updates either _MobilePhone_ or _AlternateMobilePhones_ in Azure Active Directory, the values synchronized from on-premises Active Directory will persist upon the next sync cycle. This means that any updates to these values only persist when the update is performed in on-premises Active Directory and then synchronized to Azure Active Directory.

### Enable the _BypassDirSyncOverrides_ feature:

To enable BypassDirSyncOverrides  feature use the MSOnline PowerShell module.

```powershell
Set-MsolDirSyncFeature -Feature BypassdirSyncOverrides -Enable $true
```

Once the feature is enabled, start a full synchronization cycle in Azure AD Connect using the following command:

```powershell
Start-ADSyncSyncCycle -PolicyType Initial
```

[!NOTE] Only objects with a different _MobilePhone_ or _AlternateMobilePhones_ value from on-premises Active Directory will be updated.

### Verify the status of the _BypassDirSyncOverrides_ feature:

```powershell
Get-MsolDirSyncFeatures -Feature BypassdirSyncOverrides 
```

## Disabling _BypassDirSyncOverrides_ feature

If you desire to restore the ability to update mobile phone numbers from the portal or PowerShell, you can disable _BypassDirSyncOverrides_ feature using the following Microsoft Online PowerShell module command:

```powershell
Set-MsolDirSyncFeature -Feature BypassdirSyncOverrides -Enable $false
```

When this feature is turned off, anytime a user or admin updates the _MobilePhone_ or _AlternateMobilePhones_ directly in Azure AD, a _DirSyncOverrides_ is created which prevents any future updates to these attributes coming from on-premises Active Directory. From this point on, a user or admin can only manage these attributes from Azure AD as any new updates from on-premises _Mobile_ or _OtherMobile_ will be dismissed.

## Managing mobile phone numbers in Azure AD and on-premises Active Directory

To manage the user’s phone numbers, an admin can use the following set of functions from _ADSyncTools_ module to read, write and clear the values in either Azure AD or on-premises Active Directory.

### Get _Mobile_ and _OtherMobile_ properties from on-premises Active Directory:

```powershell
Get-ADSyncToolsDirSyncOverridesUser 'User1@Contoso.com' -FromAD
```

### Get _MobilePhone_ and _AlternateMobilePhones_ properties from Azure AD:

```powershell
Get-ADSyncToolsDirSyncOverridesUser 'User1@Contoso.com' -FromAzureAD
```

### Set _MobilePhone_ and _AlternateMobilePhones_ properties in Azure AD:

```powershell
Set-ADSyncToolsDirSyncOverridesUser 'User1@Contoso.com' -MobileInAD '999888777'  -OtherMobileInAD '0987654','1234567'
```

### Set _Mobile_ and _otherMobile_ properties in on-premises Active Directory:

```powershell
Set-ADSyncToolsDirSyncOverridesUser 'User1@Contoso.com' -MobilePhoneInAAD '999888777' -AlternateMobilePhonesInAAD '0987654','1234567'
```

### Clear _MobilePhone_ and _AlternateMobilePhones_ properties in Azure AD:

```powershell
Clear-ADSyncToolsDirSyncOverridesUser 'User1@Contoso.com' -MobileInAD -OtherMobileInAD
```

### Clear _Mobile_ and _otherMobile_ properties in on-premises Active Directory:

```powershell
Clear-ADSyncToolsDirSyncOverridesUser 'User1@Contoso.com' -MobilePhoneInAAD -AlternateMobilePhonesInAAD
```

## Next Steps

Learn more about [Azure AD Connect: ADSyncTools PowerShell Module](reference-connect-adsynctools.md)
