# AMA vs Defender Coverage Workbook

## Overview

This workbook helps security and operations teams identify gaps in endpoint monitoring by correlating Microsoft Defender for Endpoint (MDE) onboarding with Azure Monitor Agent (AMA) telemetry and security-log ingestion in Microsoft Sentinel.

Use it to find devices that are missing Defender onboarding, are not reporting AMA heartbeats, or are not sending the expected Windows security events or Linux syslog data. Summary tiles, an endpoint coverage matrix, and VM/Data Collection Rule (DCR) correlation help prioritize investigation and remediation.

> [!NOTE]
> **Part of the [Sentinel Maturity Model](https://github.com/mathijsvermaat/Sentinel-Maturity)** — tiered guidance for Microsoft Sentinel data-connector onboarding, retention and detection coverage. This workbook backs the [Defender AMA Coverage walkthrough](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/procedures/defender-ama-coverage.md); record the coverage gaps it surfaces in the [assessment checklist](https://mathijsvermaat.github.io/sentinel-maturity-assessment.html).

## Key features and use cases

The full **Log Analytics (Sentinel)** mode provides:

- **Coverage analysis:** identify MDE-only, AMA-only, and combined coverage to prioritize onboarding and telemetry remediation.
- **Log-ingestion checks:** verify Windows `SecurityEvent` and Linux `Syslog` data, with last heartbeat and log timestamps for freshness.
- **Summary tiles and endpoint matrix:** review counts and investigate individual devices.
- **Interactive filtering:** select the Sentinel workspace, time range, OS platform, heartbeat status, workstation inclusion, and compliant-device exclusion.
- **VM and DCR correlation:** compare reported telemetry with AMA extension information and DCR assignments to troubleshoot collection gaps.
- **Export:** use the workbook's export options for reporting and follow-up.

For the distinction between telemetry and extension installation, see [Understanding the results](#understanding-the-results). The limited Advanced Hunting mode does not provide all of these capabilities.

## Data source modes (Log Analytics vs Advanced Hunting)

The **Data source** filter selects the coverage view in [Defender_vs_AMA.json](Defender_vs_AMA.json).

| Mode | Coverage view | When to use |
|------|---------------|-------------|
| **Log Analytics (Sentinel)** *(default)* | One coverage matrix correlates `DeviceInfo`, `Heartbeat`, `SecurityEvent`, and `Syslog`, with summary tiles and a merged DCR view. | Defender XDR data is ingested into the selected Sentinel workspace. |
| **Advanced Hunting (Defender XDR)** | Separate Defender inventory and Sentinel telemetry tables for manual comparison. | An alternative view when Defender XDR data is not ingested into Sentinel, subject to the limitations below. |

The Advanced Hunting mode in this version is the older split-table implementation, not a full native equivalent of the Log Analytics dashboard. See [Troubleshooting and limitations](#troubleshooting-and-limitations) before choosing it.

## Prerequisites

> [!WARNING]
> The full Log Analytics workbook requires Microsoft Defender XDR data to be ingested into Sentinel. Without that ingestion, do not rely on the full coverage matrix or summary. The **Data source** toggle offers a limited [Advanced Hunting view](#data-source-modes-log-analytics-vs-advanced-hunting), not equivalent functionality.

- A Microsoft Sentinel workspace.
- Defender for Endpoint integration enabled.
- `DeviceInfo`, `Heartbeat`, `SecurityEvent`, and `Syslog` available in Log Analytics for the full mode.
- Read access to the workspace and Azure resources used by the VM/DCR inventory.
- AMA deployed on the machines expected to send telemetry.
- For the limited Advanced Hunting mode: use the Microsoft Defender portal and have Defender XDR access in addition to Log Analytics read access.

## Deployment and first use

1. Open **Microsoft Sentinel Workbooks**.
2. Select **Add Workbook → Advanced Editor**.
3. Paste [Defender_vs_AMA.json](Defender_vs_AMA.json).
4. Select your Sentinel workspace and leave **Data source = Log Analytics (Sentinel)** for the full dashboard.
5. Confirm the summary, coverage matrix, and merged DCR view load, then save the workbook.

For the limited Advanced Hunting mode, open the workbook in the [Microsoft Defender portal](https://security.microsoft.com), select that mode, and edit **Table - MDE (Advanced Hunting)** to confirm **Data source = Advanced hunting**. The two tables require manual correlation as described below.

### Initial filters

- **Time range:** defaults to 7 days.
- **Exclude Workstations:** defaults to Yes; change it to include workstations. Mobile devices remain excluded.
- **Exclude Compliant Machines:** defaults to Yes (`MDE + AMA`). In the full mode, set it to No to see compliant devices too; a fully compliant selection can otherwise produce an empty matrix.
- **AMA heartbeat seen:** select All, Yes, or No to focus on telemetry reporting in the full mode.
- **OS platform:** use the text filter to narrow the results.
- **VM name:** use the DCR inventory filter to locate specific machines.

## Understanding the results

### Coverage categories in Log Analytics mode

| Category | Meaning |
|----------|---------|
| **MDE + AMA** | Onboarded to MDE and an AMA heartbeat was seen within the selected time window. |
| **MDE Only (no AMA heartbeat)** | Onboarded to MDE but no AMA heartbeat was seen within that window. This does not prove the extension is missing. |
| **AMA Only** | A heartbeat was seen, but the device did not match an onboarded MDE device in the query's scope. |

`MDE + AMA` is the workbook's combined-coverage category, not proof that every required log stream is arriving. Also check `SendsSecurityLogs`, `SendsSyslogLogs`, and the last-event timestamps.

### Heartbeat evidence versus extension installation

The first table and executive-summary tiles infer AMA presence from `Heartbeat`, not from the installed extension state:

- `HeartbeatSeen`: Yes or No, based on a heartbeat within the selected time window.
- `AMAStatus`: `Heartbeat seen` or `No AMA or No Heartbeat`.

A machine can have AMA installed but send no heartbeat because it is powered off, its network is blocked, the AMA service has stopped, or no DCR is associated.

The final **Merge - MDEvsAMA + DCR** view cross-checks telemetry with Azure Resource Graph fields such as `hasAMAExt`, `amaExtVersion`, and `amaExtState`. Use those extension properties to check actual installation state for resources visible in the inventory. For example, `hasAMAExt = Yes` with `HeartbeatSeen = No` points to an installed but non-reporting agent.

### Manual comparison in Advanced Hunting mode

Compare `DeviceKey` in **Table - MDE (Advanced Hunting)** with `AMAKey` in **Table - AMA telemetry (Log Analytics)**:

- In both tables: the device has MDE onboarding and a heartbeat.
- In the Defender table only: the device is onboarded but has no matching heartbeat in the telemetry table.
- In the telemetry table only: the device has a heartbeat but no matching device in the displayed Defender inventory.

Interpret those matches within the selected time range and filters. This mode does not compute the unified `StatusCategory`.

## Troubleshooting and limitations

| Symptom or limitation | What to check |
|-----------------------|---------------|
| Empty matrix | Check the workspace, time range, OS filter, and **Exclude Compliant Machines** setting before assuming there are no devices. |
| No heartbeat | Review the reporting window, machine power state, network connectivity, AMA service, and DCR association. Cross-check extension state in the merged view when using Log Analytics mode. |
| No security logs | Windows is checked through `SecurityEvent`; Linux through `Syslog`. Review the relevant DCR and log-ingestion path. |
| Missing tables or access failures | Check that the selected workspace contains the required tables and that you have read access. Full Log Analytics coverage also needs ingested `DeviceInfo`. |
| Incomplete OS names | Some AMA versions report only `Windows` rather than a full server OS name. A server-name filter can therefore exclude such rows. |
| Native history beyond 30 days | Defender XDR normally retains 30 days of native data. Longer history depends on retained streamed data where configured. |

**Limitations of this version's Advanced Hunting mode:**

- The two tables require manual correlation; `StatusCategory` is not computed.
- **Exclude Compliant Machines** and **AMA heartbeat seen** are not applied to that split-table view.
- The summary tiles still depend on Log Analytics data; they are not a native Advanced Hunting summary.
- The final DCR merged view is restricted to Log Analytics mode.
- The experimental **Merge - MDE + AMA (Advanced Hunting)** step is disabled.

These are limitations of this workbook implementation, not a general restriction on the capabilities of the unified Defender portal.

## How it works / technical details

### Data and correlation

| Source | Purpose |
|--------|---------|
| `DeviceInfo` | Defender onboarding and OS information |
| `Heartbeat` | Agent reporting and last heartbeat |
| `SecurityEvent` | Windows security-log ingestion |
| `Syslog` | Linux log ingestion |
| Azure Resource Graph | VM/Arc inventory, AMA extensions, and DCR associations |

In Log Analytics mode, **Table - MDEvsAMA** joins the four telemetry tables using normalized short device names. It adds the coverage classification and applies the workbook filters. The summary query produces counts, and **Merge - MDEvsAMA + DCR** combines the matrix with the ARG inventory.

### Split-table Advanced Hunting implementation

- **Table - MDE (Advanced Hunting)** queries `DeviceInfo` with `Timestamp` and `queryType: advancedHunting`, projecting `DeviceName`, `DeviceKey`, `OSPlatform`, and `MDEStatus`.
- **Table - AMA telemetry (Log Analytics)** queries `Heartbeat`, `SecurityEvent`, and `Syslog` with `TimeGenerated`, projecting the matching key, heartbeat/log indicators, and timestamps.
- **Merge - MDE + AMA (Advanced Hunting)** retains the earlier merge attempt, but its `DataSourceMode == "__MergeDisabled__"` visibility condition prevents it from being displayed. It is not part of the working coverage view.

The standalone query is available separately in [Defender_AMA_coverage.kql](Defender_AMA_coverage.kql).

## Related resources

- **[Sentinel Maturity Model](https://github.com/mathijsvermaat/Sentinel-Maturity)** — the tiered connector guidance model this workbook belongs to.
- **[Defender AMA Coverage walkthrough](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/procedures/defender-ama-coverage.md)** — step-by-step guide to deploying the workbook and interpreting the coverage gaps.
- **Connectors this workbook validates** — [Windows Security Events](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/connectors/windows-security-events.md), [Syslog for Linux](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/connectors/syslog-linux.md), [Windows Forwarded Events](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/connectors/windows-forwarded-events.md) and [Defender for Cloud](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/connectors/microsoft-defender-for-cloud.md).
- **[Assessment checklist](https://mathijsvermaat.github.io/sentinel-maturity-assessment.html)** — the *Defender vs AMA coverage* gap analysis records Both / AMA only / MDE only counts straight from this workbook.
