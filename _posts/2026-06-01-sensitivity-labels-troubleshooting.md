---
title: "Purview Sensitivity Labels - Definitive troubleshooting guide"
date: 2026-06-01 00:00:00 +0000
categories: [Purview]
tags: [microsoft-purview, sensitivity-labels, information-protection, data-security, microsoft-365, purview-implementation]
author: "Jere Haavisto"
image:
  path: /assets/img/posts/2026/06/purview-sen.png
---

I have spent the last few days on banging my head against the wall with Sensitivity Lables. I´ve searched and searched for aid to my problems without any luck. Seriously, there has been some long days and in the end the fix was simple but easily overlooked. I wanted to share my pain if someone is in the same situation.

If sensitivity labels work in Office for the web, iOS, and Teams on Windows, but do not show up in Word or Outlook on Windows, the problem is usually not the label itself. In most cases, the break is in one of these places:

- The user has a qualifying Microsoft 365 SKU, but the required sensitivity labeling service plan inside that license is disabled.
- The Windows Office client is not a supported Microsoft 365 subscription build.
- Built-in labeling is disabled by policy or the old Azure Information Protection add-in path is still in play.
- Outlook has a mailbox or policy scope problem.
- The local Office label cache under `%localappdata%\Microsoft\Office\CLP` is stale or broken.

Microsoft's own troubleshooting guidance points to the same checks: subscription activation, supported Office version, published label policy, built-in labeling enabled, and a healthy CLP cache. That is the normal path and it solves most cases.

If you are earlier in your Purview journey, start with [Microsoft Purview: What Is It and Why Do I Need It?](/posts/microsoft-purview-what-is-it-and-why-do-i-need-it/) for the platform view, [Microsoft Purview Implementation, Part 0: What to Do Before the Technical Work Starts](/posts/what-to-do-before-implementing-microsoft-purview/) for the groundwork, and [Microsoft Purview Sensitivity Labels, Part 1: How to Get Started and Reach a Fully Labeled Environment](/posts/purview-sensitivity-labels-getting-started/) for the label design basics.

My case did not follow the normal path.

Everything on the Purview side looked correct. The policy downloaded to the machine. The client-side switches were enabled. The labels still did not appear in Word or Outlook.

So this post is split into two parts:

- The normal troubleshooting path that should catch the common problems.
- The less obvious case I hit, where the user had Microsoft 365 E5 but only the Premium information protection entitlement was enabled, and Word decided manual labeling was not licensed.

## Skip to the good part

- [What this symptom usually means](#what-this-symptom-usually-means)
- [Normal troubleshooting path](#normal-troubleshooting-path)
- [What to collect for support](#what-to-collect-for-support)
- [My case](#my-case)
- [What the service desk needs to check in this case](#what-the-service-desk-needs-to-check-in-this-case)
- [Final point](#final-point)

## What this symptom usually means

If labels are already visible in Office for the web and iOS, your Purview configuration is probably mostly correct. That does not completely rule out a bad policy scope, but it does shift suspicion toward the Windows desktop client. Microsoft's [Sensitivity labels are missing in Outlook, Outlook on the web, and other Office apps](https://learn.microsoft.com/troubleshoot/microsoft-365/purview/sensitivity-labels/sensitivity-labels-missing){:target="_blank"} article points to the same split.

For Outlook specifically, Microsoft also requires the mailbox to be in Exchange Online. Sensitivity labels are not supported for mailboxes hosted on-premises.

## Normal troubleshooting path

These are the checks I would run first before touching anything more exotic.

## Step 1 - Confirm the label and policy from Security and Compliance PowerShell

Start on the service side first. If these checks fail, there is no point debugging the client yet.

The underlying admin guidance is in [Create and configure sensitivity labels and their policies](https://learn.microsoft.com/purview/create-sensitivity-labels){:target="_blank"} and [Sensitivity labels are missing in Outlook, Outlook on the web, and other Office apps](https://learn.microsoft.com/troubleshoot/microsoft-365/purview/sensitivity-labels/sensitivity-labels-missing){:target="_blank"}.

```powershell
if (-not (Get-Module ExchangeOnlineManagement -ListAvailable)) {
  Install-Module ExchangeOnlineManagement -Scope CurrentUser
}

Import-Module ExchangeOnlineManagement
Connect-IPPSSession
```

List the labels:

```powershell
Get-Label |
  Select-Object DisplayName, Name, Guid, ContentType, Disabled |
  Format-Table -AutoSize
```

Check the exact label you expect to see:

```powershell
Get-Label -Identity "Your label name" |
  Format-List DisplayName, Name, Guid, ContentType, Disabled, ParentId, Priority
```

Check the published label policies:

```powershell
Get-LabelPolicy |
  Select-Object Name, Mode, Workload, DistributionStatus |
  Format-Table -AutoSize
```

Inspect the policy that should apply to the affected user:

```powershell
Get-LabelPolicy -Identity "Your policy name" |
  Format-List Name, Mode, Workload, DistributionStatus, ExchangeLocation,
    ExchangeLocationException, ModernGroupLocation, ModernGroupLocationException
```

What you want to see:

- The label is not disabled.
- `ContentType` includes the right workload. For Outlook this must include `Email`. For Word this must include `File`.
- The policy is in `Enforce` mode.
- `DistributionStatus` is `Success`.
- The user is actually in scope for the policy assignment.
- For Outlook, the policy workload includes Exchange targeting that makes sense for the user.

If the web client already shows the labels, this step often passes. That is useful, because it narrows the issue to the Windows client quickly.

## Step 2 - Confirm the user has the correct labeling entitlement enabled

This is the part that is easy to miss.

Checking only the assigned SKU is not enough. Microsoft documents in the [Microsoft Purview Information Protection sensitivity labeling service description](https://learn.microsoft.com/office365/servicedescriptions/microsoft-365-service-descriptions/microsoft-365-tenantlevel-services-licensing-guidance/microsoft-purview-service-description#microsoft-purview-information-protection-sensitivity-labeling){:target="_blank"} that for sensitivity labeling, a Standard or Plan 1 entitlement must be assigned in addition to the Premium or Plan 2 entitlement for Information Protection for Office 365 and AIP.

That means a user can have a high-end license such as Microsoft 365 E5 and still fail manual labeling in Office if the relevant service plan is disabled inside the assigned license.

The cleanest check is usually in the Microsoft 365 admin center under the user account, **Licenses and apps**, where you can inspect the individual services under the assigned product.

If you want to inspect the assigned license details with Microsoft Graph PowerShell:

```powershell
Connect-MgGraph -Scopes User.Read.All, Organization.Read.All

Get-MgUserLicenseDetail -UserId user@contoso.com |
  ForEach-Object {
    $_.ServicePlans |
      Select-Object ServicePlanName, ProvisioningStatus
  } |
  Sort-Object ServicePlanName
```

What you want to verify:

- The user has a license that supports manual sensitivity labeling in Office apps.
- The relevant labeling service plan is enabled, not just the parent SKU.
- If your tenant uses Information Protection for Office 365 or AIP standalone plans, a Standard or Plan 1 entitlement is enabled in addition to Premium or Plan 2.

If this part is wrong, Word and Outlook can receive the policy just fine and still disable manual labeling. This is also a bit tricky since you would imagine that if you enable Premium, Standard doesn´t need to be enabled.

## Step 3 - Confirm the Windows Office client is actually supported

Microsoft only supports sensitivity labels in subscription editions of Office. Standalone perpetual editions are not enough. The baseline client requirements are documented in [Manage sensitivity labels in Office apps](https://learn.microsoft.com/purview/sensitivity-labels-office-apps){:target="_blank"}.

Check Click-to-Run details:

```powershell
$clickToRun = 'HKLM:\SOFTWARE\Microsoft\Office\ClickToRun\Configuration'

if (Test-Path $clickToRun) {
  Get-ItemProperty $clickToRun |
    Select-Object Platform, ProductReleaseIds, ClientVersionToReport, UpdateChannel |
    Format-List
} else {
  'Click-to-Run configuration not found.'
}
```

Check Office licensing state:

```powershell
$ospp = Join-Path ${env:ProgramFiles(x86)} 'Microsoft Office\root\Office16\OSPP.VBS'
if (-not (Test-Path $ospp)) {
  $ospp = Join-Path $env:ProgramFiles 'Microsoft Office\root\Office16\OSPP.VBS'
}

if (Test-Path $ospp) {
  cscript.exe //nologo $ospp /dstatus
} else {
  'OSPP.VBS not found.'
}
```

What you are looking for:

- Microsoft 365 Apps, not Office LTSC or another perpetual build.
- A supported Office version. Microsoft documents manual label support in Word, Excel, PowerPoint, and Outlook on Windows from version 1910+ on Current Channel and Monthly Enterprise Channel, and 2002+ on Semi-Annual Enterprise Channel. For the exact feature matrix, check [Minimum versions for sensitivity labels in Office apps](https://learn.microsoft.com/purview/sensitivity-labels-versions){:target="_blank"}.
- A valid subscription activation state.

If the build is unsupported or the product is perpetual, stop here and fix that before doing anything else.

## Step 4 - Check whether built-in labeling is disabled on the client

Microsoft documents two common Windows-side blockers:

- `Use the Sensitivity feature in Office to apply and view sensitivity labels` is disabled.
- `Use the Azure Information Protection add-in for sensitivity labeling` is enabled, which prevents the built-in experience from being used.

The quickest way to check the effect is to read the Office labeling registry keys.

```powershell
$registryPaths = @(
  'HKCU:\Software\Microsoft\Office\16.0\Common\Security\Labels',
  'HKCU:\Software\Policies\Microsoft\Office\16.0\Common\Security\Labels'
)

foreach ($path in $registryPaths) {
  "`n[$path]"
  if (Test-Path $path) {
    Get-ItemProperty -Path $path | Format-List *
  } else {
    'Not present'
  }
}
```

What matters most:

- If `UseOfficeForLabelling` exists and is `0`, built-in labeling is effectively off.
- If `UseOfficeForLabelling` is `1`, the built-in Office labeling path is enabled.

If you manage Office through Group Policy or Cloud Policy, also verify that the old Azure Information Protection add-in setting is not enabled.

## Step 5 - Check whether the old AIP client is still involved

The Azure Information Protection Office add-in is retired. Office desktop should use built-in labeling instead. Microsoft calls this out explicitly in [Manage sensitivity labels in Office apps](https://learn.microsoft.com/purview/sensitivity-labels-office-apps#support-for-sensitivity-label-capabilities-in-apps){:target="_blank"}.

Check for installed Information Protection clients:

```powershell
$uninstallPaths = @(
  'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*',
  'HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*'
)

Get-ItemProperty -Path $uninstallPaths -ErrorAction SilentlyContinue |
  Where-Object {
    $_.DisplayName -match 'Azure Information Protection|Microsoft Purview Information Protection'
  } |
  Select-Object DisplayName, DisplayVersion, Publisher |
  Format-Table -AutoSize
```

Interpretation:

- `Microsoft Purview Information Protection` by itself is not automatically a problem.
- Evidence of the older Azure Information Protection unified labeling client or a policy forcing the AIP add-in is a red flag.
- If `UseOfficeForLabelling=0`, the old path is still the first thing to fix even if the labels work elsewhere.

## Step 6 - Inspect the local CLP policy cache

Office desktop downloads the label policy into `%localappdata%\Microsoft\Office\CLP`. If the cache is missing, stale, or malformed, Word and Outlook can fail even while web and mobile continue to work.

Check whether the cache exists and when it last changed:

```powershell
$clpPath = Join-Path $env:LOCALAPPDATA 'Microsoft\Office\CLP'

if (Test-Path $clpPath) {
  Get-ChildItem -Path $clpPath -Force |
    Select-Object Name, Length, LastWriteTime |
    Sort-Object LastWriteTime -Descending |
    Format-Table -AutoSize
} else {
  'CLP folder not found.'
}
```

On current builds, the policy files are often stored as gzip-compressed files that contain XML when decompressed. Do not assume you will see a plain `.xml` file.

List the detailed cache contents first:

```powershell
Get-ChildItem -Path $clpPath -Force |
  Select-Object FullName, Name, Length, LastWriteTime |
  Sort-Object LastWriteTime -Descending |
  Format-List
```

What you want to see:

- A CLP file for the same Office identity that is signed in locally.
- A valid `SyncFile` XML payload after decompression.
- Labels present with sensible `contenttype` values such as `File` and `Email`.

If the decompressed XML is malformed, or if the expected labels are missing, that is a strong lead. At that point I would inspect recent label text changes and then reset the local cache.

## Step 7 - Reset the local Office label cache

Microsoft's documented reset step for desktop clients is to rename the CLP folder and let Office download it again.

```powershell
Get-Process WINWORD, EXCEL, POWERPNT, OUTLOOK -ErrorAction SilentlyContinue |
  Stop-Process -Force

$clpPath = Join-Path $env:LOCALAPPDATA 'Microsoft\Office\CLP'

if (Test-Path $clpPath) {
  $backupPath = "$clpPath.old.$(Get-Date -Format 'yyyyMMddHHmmss')"
  Rename-Item -Path $clpPath -NewName (Split-Path -Path $backupPath -Leaf)
  "Renamed to $backupPath"
} else {
  'CLP folder not found.'
}
```

Then reopen Word and Outlook, sign in normally, and wait for policy refresh.

If labels appear after this step, the root cause was local cache or local policy processing.

## Step 8 - Separate Word from Outlook if only one still fails

If Word starts showing labels but Outlook still does not, focus on Outlook-specific requirements.

Use PowerShell to confirm the mailbox is in Exchange Online:

```powershell
Connect-ExchangeOnline

Get-EXOMailbox -Identity user@contoso.com |
  Select-Object DisplayName, PrimarySmtpAddress, RecipientTypeDetails |
  Format-List
```

Things to verify:

- The mailbox is in Exchange Online.
- The label policy includes Exchange workload targeting.
- The affected mailbox is not an on-premises mailbox or a special case that falls outside supported Outlook labeling behavior.

If Outlook works in Outlook on the web but not in Outlook for Windows after the client checks above, the client-side registry and CLP steps are still the strongest suspects.

## What to collect for support

If you need to escalate this internally, collect these outputs in this order:

1. `Get-Label | Select DisplayName, Name, Guid, ContentType, Disabled`
2. `Get-LabelPolicy | Select Name, Mode, Workload, DistributionStatus`
3. `Get-LabelPolicy -Identity "Your policy name" | fl Name, Mode, Workload, DistributionStatus, ExchangeLocation, ExchangeLocationException, ModernGroupLocation, ModernGroupLocationException`
4. The user's assigned license and enabled service plans from Microsoft 365 admin center or Microsoft Graph
5. The Click-to-Run and `OSPP.VBS /dstatus` output
6. The two registry key dumps from `...\Common\Security\Labels`
7. The CLP folder listing and XML validation output
8. Any Word or Outlook CLP log lines that mention `IsClientLicensedForManualLabeling` or `DisableNativeLabeling`

That is usually enough to tell whether the problem is policy scope, unsupported Office, built-in labeling disabled, AIP conflict, or a broken local policy cache.

## Why this order works

When labels are visible in web and mobile but absent in Word and Outlook on Windows, I treat it as a desktop client problem until proven otherwise. The fastest discriminating checks are:

1. Does the user actually have the required manual labeling entitlement enabled?
2. Is this a supported Microsoft 365 Apps subscription build?
3. Is `UseOfficeForLabelling` enabled?
4. Is the old AIP path still forced somehow?
5. Does the CLP cache exist and parse cleanly?

That sequence gets to the root cause faster than repeatedly changing labels in Purview and then waiting another day for propagation.

## My case

My case turned out to be more annoying than the normal checklist.

I verified all the expected things:

- The labels were published correctly.
- Office was a supported Microsoft 365 Apps build.
- The user had Microsoft 365 E5 assigned.
- `UseOfficeForLabelling` was enabled.
- The CLP cache existed for the correct signed-in Office identity.
- After decompression, the CLP payload contained the expected labels for `File` and `Email`.
- Safe mode did not make the `Sensitivity` button appear in Word or Outlook.

At that point it no longer looked like a Purview policy problem. It looked like the desktop Office client was receiving the right policy but still refusing to allow manual labeling.

The clue came from the Word client logs which you can get by running:

```powershell
reg add HKCU\Software\Microsoft\Office\16.0\Common\Logging /v EnableLogging /t REG_DWORD /d 1
```

This creates a log file in `AppData\Local\Temp`. Run Word and open a new document.

You should run the same command again afterwards but change the end to `/d 0`, because the logs can get large quickly.

These were the important lines:

```text
IsClientLicensedForManualLabeling called. {"ClientLicensedForManualLabeling": false, "hasFullDRMPrivilege": false}
DisableNativeLabeling() called. {"DisableReason": 1}
```

That is the moment Word decided not to show the labeling UI. The policy was there, but the client believed manual labeling was not licensed.

The actual problem was not that the user lacked a high-level SKU. The user had Microsoft 365 E5 assigned. The problem was that only **Information Protection Premium** was enabled in the service plans.

Microsoft's licensing guidance calls out the trap directly: for Information Protection for Office 365 and AIP, the Standard or Plan 1 entitlement must be assigned in addition to Premium or Plan 2 for sensitivity labeling to be available. That specific note is in the [Microsoft Purview Information Protection sensitivity labeling service description](https://learn.microsoft.com/office365/servicedescriptions/microsoft-365-service-descriptions/microsoft-365-tenantlevel-services-licensing-guidance/microsoft-purview-service-description#microsoft-purview-information-protection-sensitivity-labeling){:target="_blank"}.

So the root cause in my case was simple and annoying at the same time: the user had E5, but the relevant Standard or Plan 1 labeling right was not effectively enabled for the desktop client.

## What the service desk needs to check in this case

If you get to the same point I did, stop changing labels in Purview. The next step is to inspect the actual service plans under the assigned Microsoft 365 license.

First, confirm the user license and enabled service plans in Microsoft 365 admin center or with Graph.

```powershell
Connect-MgGraph -Scopes User.Read.All, Organization.Read.All

Get-MgUserLicenseDetail -UserId user@contoso.com |
  ForEach-Object {
    $_.ServicePlans |
      Select-Object ServicePlanName, ProvisioningStatus
  } |
  Sort-Object ServicePlanName
```

Then compare that with the Word logs if you have them:

```text
IsClientLicensedForManualLabeling called. {"ClientLicensedForManualLabeling": false, "hasFullDRMPrivilege": false}
```

If Word says `ClientLicensedForManualLabeling: false`, and the user only has the Premium sensitivity labeling service enabled, you have a licensing entitlement problem rather than a missing label policy.

In my case, the fix was not more cache cleanup. It was to get the correct labeling entitlement enabled for the user.

If you still see odd activation noise on the client, that can be worth cleaning up, but it is no longer the first suspect once the Word logs tell you manual labeling is not licensed.

## Final point

If labels already work in web and mobile, resist the urge to keep redesigning your labels in Purview. Start with the ordinary desktop client checks, and make sure you inspect the actual service plans under the assigned user license.

That was the gap in my case. Purview was not broken, and the E5 SKU by itself was not enough evidence. The missing piece was that only Information Protection Premium was enabled, so Word treated manual labeling as unlicensed.

If you are still building your label model, go back to [Microsoft Purview Sensitivity Labels, Part 1: How to Get Started and Reach a Fully Labeled Environment](/posts/purview-sensitivity-labels-getting-started/). If you are still deciding whether your organization is even ready for Purview implementation work, read [Microsoft Purview Implementation, Part 0: What to Do Before the Technical Work Starts](/posts/what-to-do-before-implementing-microsoft-purview/).

This process was educational on how the labels work, how they are applied, and what affects them. I hope it helps someone else troubleshoot labels in their own environment.
