# 📊 WUfB Reports - Azure Monitor Workbooks & KQL Queries

Azure Monitor Workbooks and KQL queries for **Windows Update for Business (WUfB) Reports** on Log Analytics.

These workbooks provide visibility into Windows Update deployment status, device compliance, and update health across your organization.

## 📁 Repository Structure

```
├── workbooks/
│   ├── WUfB-Device-Update-Status.workbook
│   ├── WUfB-Driver-Compliance-Report.workbook
│   ├── WUfB-KB-Compliance-Report.workbook
│   ├── WUfB-KB-Compliance-Report-Hostname-P5.workbook
│   ├── WUfB-QualityUpdates-Compliance.workbook
│   └── WUfB-Branch-Health-Dashboard.workbook
├── kql/
│   ├── 01-device-kb-status-detail.kql
│   ├── 02-kb-compliance-summary.kql
│   ├── 03-devices-not-installed.kql
│   ├── 04-blocked-problematic-devices.kql
│   ├── 05-install-timeline.kql
│   ├── 06-state-distribution-by-category.kql
│   ├── 07-quality-updates-with-device-info.kql
│   ├── 08-active-alerts-unified.kql
│   ├── 09-delivery-optimization-by-site.kql
│   ├── 10-early-warning-kb-trend.kql
│   ├── 11-device-health-by-model.kql
│   ├── 12-alert-device-stopped-scanning.kql
│   ├── 13-alert-do-efficiency-drop.kql
│   ├── 14-alert-new-error-codes.kql
│   └── 15-country-compliance-heatmap.kql
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

### 3. WUfB - Driver Compliance Report

Driver and firmware compliance based on `UpdateCategory == "DriverUpdate"`.
Client, service, and alert records are correlated using `AzureADDeviceId` and
the driver-specific `UpdateId`. Service or alert records without `UpdateId`
are intentionally not correlated.

### 4. WUfB - KB Compliance Report

Quality-update compliance by KB, with CU month, OS, client-state, and device
filters. The Hostname P5 variant applies the additional MP5/WP5/KP5/TP5/MS5/
WS5/KS5/TS5 hostname prefixes.

Compliance percentages named `PercentInstalledReported` use only devices that
reported a status for the selected update. They must not be interpreted as the
percentage of the entire managed fleet.

### 5. WUfB - Branch/Country Health Dashboard

Security compliance gap per **Branch/Site** (derived from the `DeviceName`
hostname prefix) and per `Country`, combining a **bar chart sorted by
severity** (🟢 OK / 🟡 Warning / 🔴 Critical) — best for ranking many
sites/branches — with a **geographic map** colored by compliance gap %
(bubble size = device count) using the native `Country` field, no hostname
parsing required. A detail table with a heatmap-colored `ComplianceGapPercent`
column is also available for numeric drill-down. Adjust the Branch extraction
regex to match your own hostname naming convention.

## 🚀 Deployment

Published workbooks are immutable release points. Every deployment must create
a new workbook resource with an incremented display-name version (for example,
`WUfB - KB Compliance Report v9`) instead of updating an existing workbook.

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
$workbookVersion = "v1"

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
      displayName = "WUfB - Quality Updates Compliance $workbookVersion"
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
| 7 | Quality updates + device info | Quality update status joined with device OS/deferral config |
| 8 | Active alerts unified | `UCDeviceAlert` + `UCUpdateAlert` merged, active only |
| 9 | Delivery Optimization by site | DO bandwidth efficiency grouped by hostname-derived site/country/city |
| 10 | Early warning KB trend | Daily success/failure rate per KB, for spotting bad updates early |
| 11 | Device health by model | Failed installs joined with Intune device inventory (Model/Manufacturer) |
| 12 | Alert: device stopped scanning | Devices with `LastWUScanTime` older than 14 days |
| 13 | Alert: DO efficiency drop | Day-over-day drop > 30% in `BWOptPercent7Days` |
| 14 | Alert: new error codes | `ErrorCode` active on ≥ N devices in the last 24h |
| 15 | Country compliance heatmap | Security compliance gap % by `Country`, for the workbook Map visual |

## 🔍 Advanced Monitoring - Schema Notes and Limitations

Deep-dive requested on a few advanced scenarios (Delivery Optimization by
site, early warning, model correlation, custom alerting, Power BI, geographic
heatmap). Summary of what's natively available versus what requires
additional integrations:

| Scenario | Natively available? | Notes |
|---|---|---|
| DO efficiency by site/subnet | ⚠️ Partial | No `Site`/`Subnet` field in `UCDOStatus`. Use `City`/`Country` (IP-based geo) or derive the site from the hostname prefix (query 9). Real subnets require an external mapping table. |
| Early Warning KB trend | ✅ Yes | `UCClientUpdateStatus` as a time series (query 10). |
| Device Health by model | ⚠️ Requires external join | `UCClient.DeviceManufacturer`/`DeviceModel` are documented by Microsoft as *"currently not gathered"* (not populated). Requires joining with the Intune inventory (`AzureADDeviceId`) — query 11. |
| Custom alerting (KQL) | ✅ Yes | Implementable with Azure Monitor **Scheduled Query Rules** on queries 12/13/14 (stalled device, DO drop, new error codes). `UCUpdateAlert.ErrorCode` is populated; `UCDeviceAlert.ErrorCode` is not. |
| Power BI integration | ✅ Yes (external) | No native one-click export. Paths: Power BI "Get Data → Azure Monitor Logs" connector with the same KQL queries, or Diagnostic Settings to Storage/Event Hub for retention beyond Log Analytics. |
| Country/Branch heatmap | ✅ Yes | `UCClient.Country` is populated; the workbook **Map** visual natively resolves country code/name (query 15). `City` is not populated in `UCClient` (it is in `UCDOStatus`). |

## 📖 Reference: WUfB Enumerated Types

### ClientState

| Value | Description |
|-------|-------------|
| `Unknown` | Il device non ha ancora riportato uno stato per quella KB (scan non eseguito, KB non ancora ricevuta, o problema di reporting). Se persiste va investigato. |
| `Offering` | La KB è stata *offerta* (visibile in Windows Update) ma l'installazione non è ancora iniziata. Normale finché il device non si collega o aspetta active hour/deadline. |
| `Installing` | Il workflow di installazione è iniziato. Consultare `ClientSubstate` per il passo corrente (`DownloadStart`, `InstallStart`, `RestartRequired`, ecc.). |
| `Installed` | KB installata correttamente. |
| `Uninstalling` | Disinstallazione in corso. |
| `Uninstalled` | KB rimossa. |
| `Canceled` | Offerta annullata (policy cambiata, KB superata da una più recente, deployment annullato). |
| `OnHold` | Bloccato da *safeguard hold* Microsoft (incompatibilità nota), *pause* o *deferral* policy. |

### ClientState vs ServiceState — viste complementari

| Sorgente | Cosa rappresenta |
|---|---|
| **ClientState** (`UCClientUpdateStatus`) | Stato *lato device*: cosa sta facendo Windows Update sul PC. |
| **ServiceState** (`UCServiceUpdateStatus`) | Stato *lato servizio* (Windows Update / Autopatch / Deployment Service): se la KB è stata *offerta* al device dal cloud. Valori: `Pending`, `Offering`, `OnHold`, `Canceled`. |

È normale e atteso che le due colonne non coincidano: il servizio può essere `Offering` (sta offrendo) mentre il device è già `Installing` o `Installed` (ha completato il proprio flusso). Sono fotografie di due momenti/livelli diversi dello stesso processo.

### Alert: Active vs Resolved

- `AlertStatus = Active` → problema in corso, richiede attenzione.
- `AlertStatus = Resolved` → problema *passato già rientrato* (es. errore transitorio di connettività poi recuperato). **Non richiede azione** anche se compare l'icona di errore sull'`AlertClassification`.
- Nel workbook **KB Compliance Report** la join degli alert avviene su `AzureADDeviceId` *e* `TargetKBNumber`: gli alert mostrati appartengono alla KB della riga, non ad altre KB storiche dello stesso device.

### Casi particolari di lettura

- **Righe "vuote" nel Service-Side report** → ora la colonna `Note` riporta esplicitamente `AwaitingRestart` quando il device sta solo aspettando il riavvio, `OK - Installed`, `Offered, not yet started`, ecc.
- **Mismatch device tra report Compliance e Service-Side** → ridotto: il Service-Side ora elenca *tutti* i device del report precedente, indipendentemente dalla presenza di record in `UCServiceUpdateStatus` o `UCUpdateAlert`.
- **`-1` nelle colonne `WU*DeferralDays / DeadlineDays / GracePeriodDays`** → valore sentinel di Windows; nel workbook viene renderizzato come `NotConfigured`.
- **Filtro `Tipo PC` (Modern)** → tutti i report sono filtrati di default sui soli **PC Modern** (Windows 11). I device **Windows 10** vengono esclusi automaticamente (predicato `not(OSVersion startswith "Windows 10")`, che mantiene anche i device con `OSVersion` non popolata). Per includerli, impostare *Tipo PC = Tutti i PC*.
- **Report Service-Side e Alert (KB Compliance) e cumulative** → la join service/alert è in *leftouter* su `AzureADDeviceId` + `TargetBuild`: con *CU Month = All* vengono mostrate **tutte le cumulative**, non solo l'ultima.
- **Report "Configurazione Update Ring" rimosso** dal workbook *KB Compliance Report*: i dati di deferral risultavano non disponibili (`-1`/vuoti) e il nome device spesso assente.

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
