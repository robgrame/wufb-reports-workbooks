# 📊 WUfB Reports - Azure Monitor Workbooks & KQL Queries

Azure Monitor Workbooks and KQL queries for **Windows Update for Business (WUfB) Reports** on Log Analytics.

These workbooks provide visibility into Windows Update deployment status, device compliance, and update health across your organization.

## 📁 Repository Structure

```
├── workbooks/
│   ├── WUfB-Device-Update-Status.workbook     # General update status (all categories)
│   └── WUfB-QualityUpdates-Compliance.workbook # Quality Updates focused (Security + NonSecurity)
├── kql/
│   └── WUfB-KQL-Queries.kql                   # Standalone KQL queries for Log Analytics
├── images/
└── README.md
```

## 🎯 Workbooks

### 1. WUfB - Stato Aggiornamento Device per KB

General-purpose workbook showing update installation status for **all update categories** (Quality, Feature, Driver).

**Features:**
- 🔘 Filters: time range, update category, client state, device name
- 🥧 Pie chart: device distribution by installation state
- 📊 Bar chart: ClientState × ClientSubstate breakdown
- 📋 Detail table: per-device, per-KB installation status with icons
- 📈 KB compliance summary: % installed per KB with heatmap
- ⚠️ Problem devices: blocked, canceled, or unknown states

### 2. WUfB - Quality Updates Compliance

Focused workbook for **Quality Updates only** (Security and Non-Security), integrating data from multiple WUfB tables.

**Features:**
- 📊 Compliance overview tiles from `UCClient`
- 🥧 Security Update Status + Quality Update Status pie charts
- 📋 Per-device/KB detail with joined device info (OS version, deferral config, pause state)
- 📈 KB compliance summary with % installed heatmap
- 🚦 Service-side status from `UCServiceUpdateStatus`
- ⚠️ Update & Device alerts from `UCUpdateAlert` + `UCDeviceAlert`
- 🖥️ Device configuration: WU deferral days, deadline, grace period, pause state

**Tables Used:**

| Table | Purpose |
|-------|---------|
| `UCClientUpdateStatus` | Per-device, per-update installation state (client-side) |
| `UCClient` | Device info, OS version, WU configuration, compliance status |
| `UCServiceUpdateStatus` | Service-side offering state (Autopatch/WUfB deployment service) |
| `UCUpdateAlert` | Alerts for specific update issues (connectivity, WU errors, etc.) |
| `UCDeviceAlert` | Device-level alerts (EndOfService, missing updates, diagnostics) |

## 🚀 Deployment

### Option 1: Import via Azure Portal (UI)

1. Go to **Azure Portal** → your **Log Analytics workspace**
2. Navigate to **Workbooks** → **+ New**
3. Click the **Advanced Editor** icon (`</>`)
4. Paste the content of the `.workbook` file
5. Click **Apply** → **Save**

### Option 2: Deploy via Azure CLI (ARM Template)

```powershell
# Variables
$resourceGroup = "rg-wufb-reports"
$workspaceName = "law-wufb-reports"
$workbookFile = "workbooks/WUfB-QualityUpdates-Compliance.workbook"

# Get workspace ID
$workspaceId = az monitor log-analytics workspace show `
  --workspace-name $workspaceName `
  --resource-group $resourceGroup `
  --query "id" -o tsv

# Read and prepare workbook content
$content = Get-Content $workbookFile -Raw
$serialized = $content | ConvertFrom-Json | ConvertTo-Json -Depth 50 -Compress
$workbookId = [guid]::NewGuid().ToString()

# Create ARM template
$arm = @{
  '$schema' = "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#"
  contentVersion = "1.0.0.0"
  resources = @(@{
    type = "Microsoft.Insights/workbooks"
    apiVersion = "2022-04-01"
    name = $workbookId
    location = "westeurope"  # Change to your region
    kind = "shared"
    properties = @{
      displayName = "WUfB - Quality Updates Compliance"
      serializedData = $serialized
      version = "1.0"
      sourceId = $workspaceId
      category = "workbook"
    }
  })
} | ConvertTo-Json -Depth 10

$arm | Out-File "deploy.json" -Encoding UTF8

# Deploy
az deployment group create `
  --resource-group $resourceGroup `
  --template-file "deploy.json"
```

## 📝 KQL Queries

The `kql/` folder contains standalone queries you can run directly in Log Analytics:

| # | Query | Description |
|---|-------|-------------|
| 1 | Device/KB detail | Latest installation state per device per KB |
| 2 | KB compliance summary | % installed, installing, offering per KB |
| 3 | Not installed | Devices where update is not yet installed |
| 4 | Blocked/problematic | Devices with OnHold, Canceled, SafeguardHold |
| 5 | Install timeline | Average time from offer to install per KB |
| 6 | Category distribution | State distribution per update category |

## 📖 Reference: WUfB Enumerated Types

### ClientState

| Value | Description |
|-------|-------------|
| `Unknown` | No client data available |
| `Offering` | Update offered to device |
| `Installing` | Installation in progress |
| `Installed` | Update successfully installed |
| `Uninstalling` | Uninstall in progress |
| `Uninstalled` | Update removed |
| `Canceled` | Update canceled |
| `OnHold` | Update on hold |

### UpdateCategory

| Value | Description |
|-------|-------------|
| `WindowsQualityUpdate` | Quality update (security/non-security) |
| `WindowsFeatureUpdate` | Feature update |
| `DriverUpdate` | Driver update |

### UpdateClassification

| Value | Description |
|-------|-------------|
| `Security` | Quality update with security fixes |
| `NonSecurity` | Quality update without security fixes |
| `Upgrade` | Feature update |

### ServiceState

| Value | Description |
|-------|-------------|
| `Pending` | Update not ready to be offered |
| `Offering` | Update available via Windows Update |
| `OnHold` | Offering held indefinitely |
| `Canceled` | Offering canceled |

### OSSecurityUpdateStatus

| Value | Description |
|-------|-------------|
| `Latest` | Device has latest security update |
| `NotLatest` | Device is missing the latest security update |
| `MultipleSecurityUpdatesMissing` | Device is missing multiple security updates |

> Full reference: [WUfB Reports Schema - Enumerated Types](https://learn.microsoft.com/en-us/windows/deployment/update/wufb-reports-schema-enumerated-types)

## 📋 Prerequisites

- Azure subscription with a Log Analytics workspace
- [Windows Update for Business reports](https://learn.microsoft.com/en-us/windows/deployment/update/wufb-reports-overview) configured and collecting data
- Devices enrolled in WUfB reporting (diagnostic data level: Required or higher)

## 🏷️ Topics

`azure` `log-analytics` `kql` `azure-monitor` `workbooks` `windows-update` `wufb` `wufb-reports` `quality-updates` `security-updates` `compliance` `intune` `windows-autopatch` `endpoint-management`

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🤝 Contributing

Contributions are welcome! Feel free to:
- Submit issues for bugs or feature requests
- Open pull requests with improvements
- Share additional KQL queries or workbook variations

---

*Built with ❤️ for IT admins managing Windows Update compliance at scale.*
