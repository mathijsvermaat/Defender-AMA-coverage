***

# AMA vs Defender Coverage Workbooks

> [!NOTE]
> **Part of the [Sentinel Maturity Model](https://github.com/mathijsvermaat/Sentinel-Maturity)** — tiered guidance for Microsoft Sentinel data-connector onboarding, retention and detection coverage. This workbook backs the [Defender AMA Coverage walkthrough](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/procedures/defender-ama-coverage.md); record the coverage gaps it surfaces in the [assessment checklist](https://mathijsvermaat.github.io/sentinel-maturity-assessment.html).

## Choose a workbook

| File | Status | Data sources |
|------|--------|--------------|
| [Defender_vs_AMA.json](Defender_vs_AMA.json) | Existing workbook / fallback; unchanged by the native XDR addition | Use **Log Analytics (Sentinel)** mode with `DeviceInfo`, `Heartbeat`, `SecurityEvent`, and `Syslog` in the Sentinel workspace. |
| [Defender_vs_AMA_NativeXDR.json](Defender_vs_AMA_NativeXDR.json) | **Preview / opt-in** | Native Defender XDR `DeviceInfo` plus `Heartbeat`, `SecurityEvent`, and `Syslog` from the selected Sentinel workspace. No Defender data ingestion into Log Analytics is required. |

The native XDR preview has been validated in one test tenant, but has not yet been broadly validated in production. Import it as a **separate workbook** and keep your existing deployment as a fallback. Importing the preview does not replace the existing workbook or change data ingestion configuration.

> [!WARNING]
> The native XDR preview runs in the Microsoft Defender portal and requires a Microsoft Sentinel workspace connected to the unified experience. The existing Log Analytics workbook remains available for environments that ingest Defender XDR data into Sentinel.

When running the KQL query, the **AMA presence** in the first table is inferred from the `Heartbeat` table within the selected time window — not from the actual extension state. The reason is that the real installation state is only available via an Azure Resource Graph (ARG) call. As a result, a device may show as `No AMA or No Heartbeat` / `MDE Only (no AMA heartbeat)` even when the AMA extension is installed but not reporting (for example: VM powered off, network blocked, AMA service stopped, or no DCR associated).

To make this explicit, the query and workbook expose two separate columns:

- `HeartbeatSeen` — `Yes` / `No`, based purely on the `Heartbeat` table
- `AMAStatus` — `Heartbeat seen` or `No AMA or No Heartbeat`

The **merged view** at the bottom of the workbook (`Merge - MDEvsAMA + DCR`) cross-checks this with `hasAMAExt` / `amaExtVersion` from Azure Resource Graph and is the authoritative source for whether the AMA extension is actually installed.

## Overview

This Microsoft Sentinel Workbook provides visibility into Microsoft Defender for Endpoint (MDE)–managed devices and their telemetry coverage within Sentinel. It helps security and operations teams verify that devices are properly configured for comprehensive monitoring by checking:

*   **Azure Monitor Agent (AMA)** installation status
*   **SecurityEvent** log ingestion into Sentinel (Windows)
*   **Syslog** log ingestion into Sentinel (Linux)
*   **Last heartbeat and log timestamps** for freshness

By correlating data from **DeviceInfo**, **Heartbeat**, and **SecurityEvent/Syslog** tables, the workbook identifies configuration gaps and supports remediation efforts.

***

## Data sources

### Existing workbook

[Defender_vs_AMA.json](Defender_vs_AMA.json) retains the implementation from `main` without changes. Its full coverage matrix and summary use Log Analytics and require ingested `DeviceInfo` data. Keep its **Data source** setting on **Log Analytics (Sentinel)** for this fallback path. Its older, limited Advanced Hunting mode is also unchanged; use the separate preview file for the new full native implementation.

### Native XDR preview

The coverage query in [Defender_vs_AMA_NativeXDR.json](Defender_vs_AMA_NativeXDR.json) uses the **Advanced Hunting** workbook data source in the unified Microsoft Defender portal. This allows one KQL query to correlate:

- Native Defender XDR `DeviceInfo` data, using its `Timestamp` field
- Microsoft Sentinel `Heartbeat`, `SecurityEvent`, and `Syslog` data, using `TimeGenerated`

The existing filters, executive summary tiles, endpoint coverage matrix, and merged DCR view all use this unified result. Azure Resource Graph remains the source for VM extension and DCR association details.

The connected Sentinel workspace controls which Sentinel tables are available. Users need access to the Defender XDR data and at least the Microsoft Sentinel Reader role. See [Advanced hunting with Microsoft Sentinel data in the Microsoft Defender portal](https://learn.microsoft.com/defender-xdr/advanced-hunting-microsoft-defender).

The existing **Workspace** picker selects an Azure resource ID. A hidden, query-backed `WorkspaceId` parameter resolves its Log Analytics customer GUID through Azure Resource Graph and supplies it to the Advanced Hunting `selectedWorkspaceIds` setting. `crossComponentResources` alone does not scope Advanced Hunting queries. Selecting a workspace is required; the coverage queries wait until its GUID is resolved rather than silently querying a default workspace.

The distributed workbook has **no prefilled workspace or workspace GUID**. Defender XDR data comes from the signed-in Defender tenant; the workspace picker scopes only the Sentinel data. GUIDs in `id`, `key`, and merge identifiers are internal workbook control IDs, not tenant or workspace bindings. They do not need to be changed for another tenant.

The portal can include saved selections and cached parameter values when exporting an already configured workbook. Use the repository JSON as the portable template. Before sharing an edited/exported copy, remove saved workspace selections and cached GUID values, and retain the parameter references in the query scopes rather than literal tenant-specific IDs.

In the AMA-only branch, Sentinel telemetry joins run **before** the Defender exclusion and onboarding-status join. This preserves the existing results while avoiding the HTTP 500 (`ErrorCode: 9`) reproduced when a Sentinel join followed those cross-source operations. The same query order is used by the matrix and summary tiles.

***

## Key Features

*   **Coverage Analysis**
    Detect devices that:
    *   Are onboarded to MDE but missing an AMA heartbeat (potentially missing AMA, or installed but not reporting)
    *   Are not sending SecurityEvent/Syslog logs despite being onboarded

    > **Note:** AMA presence in the first table is determined by the `Heartbeat` table only. See the [Important Notes](#important-notes) section for how to interpret `No AMA or No Heartbeat`.

*   **Filtering Options**
    Filter by:
    *   Workspace
    *   Time range (default: 7 days)
    *   OS platform
    *   AMA status (All, Yes, No) — based on whether an AMA heartbeat was seen in the time window
    *   Exclude Workstations (default: Yes)
    *   Exclude Compliant Machines

*   **Summary Tiles**
    Quick overview of device counts based on AMA status

*   **Detailed Breakdown**
    Categorizes devices as:
    *   **MDE + AMA**
    *   **MDE Only (no AMA heartbeat)**
    *   **AMA Only**

*   **DCR Association**
    Displays Data Collection Rules (DCRs) linked to machines for AMA configuration

*   **Merged View**
    Combines AMA-enabled and/or Defender devices with associated DCRs for full visibility

***

## Important Notes

*   **AMA presence is heartbeat-based in the first table**
    The first table and the executive-summary tiles classify AMA presence using the `Heartbeat` table. A `No` / `No AMA or No Heartbeat` result does **not** prove that the AMA extension is uninstalled — it only means no heartbeat was received in the selected time window. Common causes for a missing heartbeat while the extension is installed:
    - VM is powered off or deallocated
    - Network connectivity to AMA endpoints is blocked
    - AMA service is stopped or misconfigured
    - No Data Collection Rule (DCR) is associated with the machine

    The **merged view** at the bottom of the workbook joins this with Azure Resource Graph (`hasAMAExt`, `amaExtVersion`, `amaExtState`) and is the authoritative source for the actual extension installation state.

*   **Windows and Linux Support**
    This workbook supports both **Windows** and **Linux** endpoints.
    - Windows devices are validated using the **SecurityEvent** table
    - Linux devices are validated using the **Syslog** table

*   **Log Ingestion Check**
    Queries validate security log ingestion into Sentinel using **SecurityEvent** (Windows) and **Syslog** (Linux) tables.

*   **Device Type Filtering**
    By default, workstations and mobile devices are excluded to focus on server infrastructure. This can be toggled via the **Exclude Workstations** filter.

*   **OS Name Limitation**
    Some AMA versions do not report the full OS name (e.g., only `Windows` instead of `Windows Server 2025`).
    This can make filtering by server OS more challenging. Consider using additional metadata or naming conventions for accurate filtering.

***

## How It Works

1.  Collects data from:
    *   **DeviceInfo** (Defender onboarding status)
    *   **Heartbeat** (AMA presence and last seen timestamp)
    *   **SecurityEvent** (Windows security log ingestion)
    *   **Syslog** (Linux security log ingestion)
2.  Joins and correlates AMA presence, Defender onboarding, and log ingestion.
3.  Applies filters for AMA status and OS platform.
4.  Outputs:
    *   Interactive tiles for quick insights
    *   Detailed tables for device-level analysis
    *   Export options for Excel

***

## Use Cases

*   Validate AMA deployment across Defender-managed endpoints
*   Ensure SecurityEvent or Syslog logs are flowing into Sentinel
*   Identify gaps in telemetry for compliance and security posture
*   Correlate AMA coverage with DCR assignments for troubleshooting

***

## Prerequisites

### Existing workbook

*   A Microsoft Sentinel workspace with `DeviceInfo`, `Heartbeat`, `SecurityEvent`, and `Syslog` available in Log Analytics
*   Read access to the workspace and Azure resources used by the VM/DCR inventory

### Native XDR preview

*   Microsoft Sentinel workspace connected to the unified Microsoft Defender portal
*   Defender for Endpoint access
*   Microsoft Sentinel Reader role or higher
*   Azure resource read access for the workspace GUID lookup and the existing VM/DCR inventory
*   AMA deployed on target machines
*   Native Defender XDR table:
    *   `DeviceInfo`
*   Relevant tables in the connected Sentinel workspace:
    *   `Heartbeat`
    *   `SecurityEvent`
    *   `Syslog`

***

## Deployment

### Existing workbook / fallback

If already deployed, continue using the existing workbook without changes. For a new deployment, open **Microsoft Sentinel Workbooks**, select **Add Workbook → Advanced Editor**, and paste [Defender_vs_AMA.json](Defender_vs_AMA.json). Select your workspace and leave **Data source = Log Analytics (Sentinel)**.

### Native XDR preview

1.  Open **Microsoft Sentinel → Workbooks** in the [Microsoft Defender portal](https://security.microsoft.com)
2.  Click **Add Workbook → Advanced Editor**
3.  Paste [Defender_vs_AMA_NativeXDR.json](Defender_vs_AMA_NativeXDR.json)
4.  Select the connected Sentinel workspace using the workbook's **Workspace** filter
5.  Confirm the summary, coverage matrix, and merged DCR view load
6.  Save under a separate name, such as **AMA vs Defender - Native XDR Preview**; do not overwrite your existing workbook
7.  Validate the expected coverage with your own data and filters before adopting the preview in production

If the preview fails or its results are unexpected, continue using the existing workbook. Keep that deployment available until the preview has been validated in your environment.

**Advisory:**
- Default time range is 7 days (adjustable)
- Native Defender XDR data is normally retained for 30 days. Selecting a longer range does not create additional native history.
- Workstations are excluded by default (toggle with **Exclude Workstations** filter)
- By default, the **Exclude Compliant** filter is set to `MDE + AMA`, which excludes compliant machines so you can focus on remediation. Adjust this filter to include compliant devices if needed.
- A `WorkspaceId` parameter message means the workspace is not yet selected or its Azure Resource Graph lookup has not resolved. Check the top-level workspace selection and Azure read permissions.
- Select a workspace connected to the Defender portal that exposes the required Sentinel tables. Missing tables or access failures are not treated as zero telemetry.
- The standalone [Defender_AMA_coverage.kql](Defender_AMA_coverage.kql) file is unchanged by this workbook migration.

***

## Related

- **[Sentinel Maturity Model](https://github.com/mathijsvermaat/Sentinel-Maturity)** — the tiered connector guidance model this workbook belongs to.
- **[Defender AMA Coverage walkthrough](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/procedures/defender-ama-coverage.md)** — step-by-step guide to deploying the workbook and interpreting the coverage gaps.
- **Connectors this workbook validates** — [Windows Security Events](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/connectors/windows-security-events.md), [Syslog for Linux](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/connectors/syslog-linux.md), [Windows Forwarded Events](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/connectors/windows-forwarded-events.md) and [Defender for Cloud](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/connectors/microsoft-defender-for-cloud.md).
- **[Assessment checklist](https://mathijsvermaat.github.io/sentinel-maturity-assessment.html)** — the *Defender vs AMA coverage* gap analysis records Both / AMA only / MDE only counts straight from this workbook.

***
