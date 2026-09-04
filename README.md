***

# AMA vs Defender Coverage Workbook

> [!NOTE]
> **Part of the [Sentinel Maturity Model](https://github.com/mathijsvermaat/Sentinel-Maturity)** — tiered guidance for Microsoft Sentinel data-connector onboarding, retention and detection coverage. This workbook backs the [Defender AMA Coverage walkthrough](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/procedures/defender-ama-coverage.md); record the coverage gaps it surfaces in the [assessment checklist](https://mathijsvermaat.github.io/sentinel-maturity-assessment.html).

#### ⚠️ This workbook assumes Microsoft Defender XDR data is ingested into Sentinel. Without ingestion, device name normalization and correlation may be inconsistent. To work around that, use the **Data source** toggle (see [Data source modes](#data-source-modes-log-analytics-vs-advanced-hunting)) to switch the coverage table to **Advanced Hunting**, or copy the KQL query from the GitHub page and run it in Advanced Hunting in the Defender Portal (https://security.microsoft.com). 

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

## Data source modes (Log Analytics vs Advanced Hunting)

The **Data source** filter at the top of the workbook controls how the *Endpoint Coverage Matrix* is sourced. The rest of the workbook (tiles, DCR inventory, merged DCR view) always runs against Log Analytics.

| Mode | Coverage matrix structure | When to use |
|------|---------------------------|-------------|
| **Log Analytics (Sentinel)** *(default)* | A single `Table - MDEvsAMA` query joins `DeviceInfo`, `Heartbeat`, `SecurityEvent`, and `Syslog` in one Log Analytics query, with a computed `StatusCategory`. | Defender XDR data is ingested into Sentinel. Full functionality, including tiles and the merged DCR view. |
| **Advanced Hunting (Defender XDR)** | Two side-by-side tables (see below). | Defender XDR data is **not** ingested into Sentinel. Advanced Hunting can only reach Defender XDR tables, so onboarding and AMA telemetry are fetched separately and shown side by side. |

### Why Advanced Hunting mode uses two tables

Advanced Hunting (in the Defender portal) can query Defender XDR tables but **not** the Microsoft Sentinel tables (`Heartbeat` / `SecurityEvent` / `Syslog`). Running the full single-query coverage matrix against the Advanced Hunting data source therefore fails. These are two separate data planes that cannot be joined — neither in KQL (Advanced Hunting can't see the Sentinel tables) nor by the workbook **Merge** control (the Merge data source cannot consume an `advancedHunting` query as an input). Advanced Hunting mode therefore presents the data as two correlated tables:

1. **`Table - MDE (Advanced Hunting)`** — `DeviceInfo` onboarding status from Defender XDR (`Timestamp`, `queryType: advancedHunting`). Projects `DeviceName`, `DeviceKey`, `OSPlatform`, `MDEStatus`.
2. **`Table - AMA telemetry (Log Analytics)`** — `Heartbeat` + `SecurityEvent` + `Syslog` per device from the Sentinel workspace (`TimeGenerated`). Projects `AMADevice`, `AMAKey`, `HeartbeatSeen`, `SendsSecurityLogs`, `SendsSyslogLogs`, and timestamps.

Both tables are shown. **Correlate manually on the short device name:** `DeviceKey` (top table) matches `AMAKey` (bottom table).

- A device in **both** tables = MDE + AMA.
- A device in the **top table only** = onboarded to MDE but **no** AMA heartbeat in the time window (MDE-only / not reporting).
- A device in the **bottom table only** = has an AMA heartbeat but is **not** onboarded to MDE (AMA-only).

> The `Merge - MDE + AMA (Advanced Hunting)` step is left in the workbook but disabled (a `DataSourceMode` value that never matches), because the workbook Merge control returns no rows when one of its inputs is an Advanced Hunting query.

**Setup for Advanced Hunting mode:** after importing the workbook in the Microsoft Defender portal, switch to edit mode, open the `Table - MDE (Advanced Hunting)` step, and confirm **Data source = Advanced hunting** (binding `queryType: advancedHunting`).

**Limitations of Advanced Hunting mode:**
- `StatusCategory` and the `Exclude Compliant` / `AMA heartbeat seen` filters are **not** applied (those depend on a single combined query). Use `MDEStatus` + `HeartbeatSeen` across the two tables instead.
- The Executive Summary **tiles** and the **merged DCR view** are Log-Analytics-only and hidden in Advanced Hunting mode.
- Advanced Hunting retains Defender XDR data for **30 days** by default (longer only if streamed through Sentinel).
- Viewers need Defender XDR access in addition to Log Analytics reader.

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

*   Microsoft Sentinel workspace
*   Defender for Endpoint integration enabled
*   AMA deployed on target machines
*   Relevant tables in Log Analytics:
    *   `DeviceInfo`
    *   `Heartbeat`
    *   `SecurityEvent`
    *   `Syslog`

***

## Deployment

1.  Open **Microsoft Sentinel Workbooks**
2.  Click **Add Workbook → Advanced Editor**
3.  Paste the JSON from this repository
4.  Save and customize filters as needed
5.  *(Optional)* To use **Advanced Hunting** mode, open the workbook in the Microsoft Defender portal and confirm the `Table - MDE (Advanced Hunting)` item has **Data source = Advanced hunting**. See [Data source modes](#data-source-modes-log-analytics-vs-advanced-hunting).

**Advisory:**
- Default time range is 7 days (adjustable)
- Workstations are excluded by default (toggle with **Exclude Workstations** filter)
- By default, the **Exclude Compliant** filter is set to `MDE + AMA`, which excludes compliant machines so you can focus on remediation. Adjust this filter to include compliant devices if needed.

***

## Related

- **[Sentinel Maturity Model](https://github.com/mathijsvermaat/Sentinel-Maturity)** — the tiered connector guidance model this workbook belongs to.
- **[Defender AMA Coverage walkthrough](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/procedures/defender-ama-coverage.md)** — step-by-step guide to deploying the workbook and interpreting the coverage gaps.
- **Connectors this workbook validates** — [Windows Security Events](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/connectors/windows-security-events.md), [Syslog for Linux](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/connectors/syslog-linux.md), [Windows Forwarded Events](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/connectors/windows-forwarded-events.md) and [Defender for Cloud](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/connectors/microsoft-defender-for-cloud.md).
- **[Assessment checklist](https://mathijsvermaat.github.io/sentinel-maturity-assessment.html)** — the *Defender vs AMA coverage* gap analysis records Both / AMA only / MDE only counts straight from this workbook.

***
