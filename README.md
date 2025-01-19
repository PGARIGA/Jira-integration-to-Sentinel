# Integrating Jira Logs into Microsoft Sentinel Using Azure Functions

## Introduction
Monitoring and analyzing audit logs is critical for maintaining the security and compliance of your IT infrastructure. Atlassian Jira provides auditing logs, but integrating these logs into Microsoft Sentinel can enhance your ability to detect and respond to potential threats. This guide will walk you through creating an Azure Function that fetches Jira audit logs and sends them to Microsoft Sentinel.

---

## Prerequisites
Before you begin, ensure you have the following:

1. **Microsoft Azure Account:** An active subscription with Microsoft Sentinel set up.
2. **Jira Access:** Admin access to your Jira instance to fetch audit logs via REST API.
3. **Azure Functions Core Tools:** Installed on your local machine for deployment.
4. **PowerShell Knowledge:** Familiarity with PowerShell scripting.
5. **Sentinel Workspace Details:**
   - Workspace ID
   - Shared Key (Primary Key)
6. **Jira API Credentials:**
   - Email
   - API Token

---

## Steps to Integrate Jira Logs with Microsoft Sentinel

### Step 1: Create an Azure Function App

1. **Log in to the Azure portal.**
2. **Create a new Function App:**
   - Choose Runtime Stack as **PowerShell Core**.
   - Deploy your app to the desired region and resource group.

### Step 2: Create the PowerShell Script

Create a file named `atlassian.ps1` and paste the following code:

```powershell
# Input bindings are passed in via param block.
param($mytimer)

# Parameters
$WorkspaceId = "Your-Workspace-ID"
$SharedKey = "Your-Shared-Key"
$LogType = "JiraAuditsLogs"
$AtlassianUrl = "https://your-jira-instance.atlassian.net/rest/api/3/auditing/record"
$AuthEmail = "Your-Email"
$AuthToken = "Your-Jira-API-Token"

# Fetch current UTC time
$currentUTCtime = (Get-Date).ToUniversalTime()

# Timer trigger check
if ($mytimer.IsPastDue) {
    Write-Host "PowerShell timer is running late!"
}

# Calculate the time range for the last 5 minutes
$EndTime = (Get-Date).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
$StartTime = (Get-Date).AddMinutes(-5).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")

Write-Host "Fetching logs from Jira between $StartTime and $EndTime..."

# Fetch Jira logs
try {
    $Headers = @{
        "Accept" = "application/json"
        "Authorization" = "Basic " + [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("${AuthEmail}:${AuthToken}"))
    }
    $Url = "${AtlassianUrl}?from=${StartTime}&to=${EndTime}&limit=100"
    $Response = Invoke-RestMethod -Uri $Url -Headers $Headers -ErrorAction Stop
    Write-Host "Logs fetched successfully."

    # Parse logs
    $ParsedLogs = $Response.records | ForEach-Object {
        [PSCustomObject]@{
            LogEntryID   = $_.id
            Summary      = $_.summary
            Created      = $_.created
            Author       = $_.authorKey
            RemoteIP     = $_.remoteAddress
        }
    }

    # Send logs to Sentinel
    Write-Host "Sending logs to Microsoft Sentinel..."
    $Body = $ParsedLogs | ConvertTo-Json -Depth 10
    $ContentLength = $Body.Length
    $Date = (Get-Date -Format "R")

    Function Build-Signature {
        param ([string]$Date, [int]$ContentLength)
        $StringToHash = "POST`n$ContentLength`napplication/json`nx-ms-date:$Date`n/api/logs"
        $HMACKey = [System.Convert]::FromBase64String($SharedKey)
        $BytesToHash = [System.Text.Encoding]::UTF8.GetBytes($StringToHash)
        $Hash = New-Object System.Security.Cryptography.HMACSHA256
        $Hash.Key = $HMACKey
        [Convert]::ToBase64String($Hash.ComputeHash($BytesToHash))
    }

    $Signature = "SharedKey ${WorkspaceId}:" + (Build-Signature -Date $Date -ContentLength $ContentLength)
    $SentinelUri = "https://${WorkspaceId}.ods.opinsights.azure.com/api/logs?api-version=2016-04-01"
    $SentinelHeaders = @{
        "Content-Type" = "application/json"
        "Authorization" = $Signature
        "Log-Type" = $LogType
        "x-ms-date" = $Date
    }

    Invoke-RestMethod -Uri $SentinelUri -Method Post -Headers $SentinelHeaders -Body $Body -ErrorAction Stop
    Write-Host "Logs successfully sent to Sentinel."
} catch {
    Write-Error "Error: $_"
}
```

### Step 3: Configure `function.json`

Create a `function.json` file in the same directory with the following content:

```json
{
  "scriptFile": "atlassian.ps1",
  "bindings": [
    {
      "name": "mytimer",
      "type": "timerTrigger",
      "direction": "in",
      "schedule": "0 */5 * * * *"
    }
  ]
}
```

### Step 4: Deploy the Azure Function

1. **Set Up an Azure Function App:**
   - Log in to the Azure Portal.
   - Search for "Function App" in the Azure search bar and click Create.
   - Fill in the required details:
     - **Subscription:** Choose your subscription.
     - **Resource Group:** Create a new one or select an existing group.
     - **Function App Name:** Enter a unique name.
     - **Region:** Select the closest region.
     - **Runtime Stack:** Choose **PowerShell Core**.
     - **Plan Type:** Choose **Consumption Plan** (pay-as-you-go).
   - Enable monitoring with Application Insights.
   - Review and click Create.

2. **Set Up a Timer-Triggered Function:**
   - Navigate to your Function App.
   - Click Functions > Create > Timer Trigger.
   - Set the Schedule to `0 */5 * * * *` for a 5-minute interval.

3. **Upload the PowerShell Script:**
   - Go to Code + Test in the left menu.
   - Delete the default script and paste the `atlassian.ps1` script.
   - Save the script.

4. **Configure `function.json`:**
   - Ensure the Timer Trigger is configured correctly.
   - Edit and save the `function.json` file.

### Step 5: Test the Function

1. **Manually Trigger the Function:**
   - Go to the Azure Portal.
   - Select your Function App and navigate to the function.
   - Click Test/Run and select Run.

2. **Check the Logs:**
   - Go to the Monitor section to view execution logs.
   - Verify that logs are fetched from Jira and sent to Sentinel.

### Step 6: Verify Data in Microsoft Sentinel

1. **Open Microsoft Sentinel:**
   - Navigate to your Sentinel workspace in Azure.

2. **Check Incoming Data:**
   - Open the Logs tab and run the following query:

```kql
JiraAuditsLogs
| limit 10
```

---

## Notes

- Update the placeholder details in the `atlassian.ps1` script with actual values:

| Key            | Value                                      |
|----------------|--------------------------------------------|
| WorkspaceId    | Your Sentinel Workspace ID                |
| SharedKey      | Your Sentinel Shared Key                  |
| AtlassianUrl   | Jira API URL                              |
| AuthEmail      | Your Jira email                           |
| AuthToken      | Your Jira API token                       |

- Ensure proper permissions are granted in both Jira and Azure for seamless integration.

---

By following these steps, you can effectively integrate Jira audit logs into Microsoft Sentinel, enhancing your monitoring and incident response capabilities.

