# AMA vs Defender Coverage Workbooks

## Overview

This workbook helps security and operations teams identify gaps in endpoint monitoring by correlating Microsoft Defender for Endpoint (MDE) onboarding with Azure Monitor Agent (AMA) telemetry and security-log ingestion in Microsoft Sentinel.

Use it to find devices that are missing Defender onboarding, are not reporting AMA heartbeats, or are not sending the expected Windows security events or Linux syslog data. Summary tiles, an endpoint coverage matrix, and VM/Data Collection Rule (DCR) correlation help prioritize investigation and remediation.

> [!NOTE]
> **Part of the [Sentinel Maturity Model](https://github.com/mathijsvermaat/Sentinel-Maturity)** — tiered guidance for Microsoft Sentinel data-connector onboarding, retention and detection coverage. This workbook backs the [Defender AMA Coverage walkthrough](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/procedures/defender-ama-coverage.md); record the coverage gaps it surfaces in the [assessment checklist](https://mathijsvermaat.github.io/sentinel-maturity-assessment.html).

## Key features and use cases

- **Coverage analysis:** identify MDE-only, AMA-only, and combined coverage to prioritize onboarding and telemetry remediation.
- **Log-ingestion checks:** verify Windows `SecurityEvent` and Linux `Syslog` data, with last heartbeat and log timestamps for freshness.
- **Summary tiles and endpoint matrix:** review counts and investigate individual devices.
- **Interactive filtering:** select the Sentinel workspace, time range, OS platform, heartbeat status, workstation inclusion, and compliant-device exclusion.
- **VM and DCR correlation:** compare reported telemetry with AMA extension information and DCR assignments to troubleshoot collection gaps.
- **Export:** use the workbook's export options for reporting and follow-up.

For the distinction between telemetry and extension installation, see [Understanding the results](#understanding-the-results).

## Choose a workbook

| File | Status | Data sources |
|------|--------|--------------|
| [Defender_vs_AMA.json](Defender_vs_AMA.json) | Existing workbook / fallback; unchanged by the native XDR addition | Use **Log Analytics (Sentinel)** mode with `DeviceInfo`, `Heartbeat`, `SecurityEvent`, and `Syslog` in the Sentinel workspace. |
| [Defender_vs_AMA_NativeXDR.json](Defender_vs_AMA_NativeXDR.json) | **Preview / opt-in** | Native Defender XDR `DeviceInfo` plus `Heartbeat`, `SecurityEvent`, and `Syslog` from the selected Sentinel workspace. No Defender data ingestion into Log Analytics is required. |

> [!WARNING]
> The native XDR preview has not yet been broadly validated in production. Import it as a **separate workbook** and keep your existing deployment as a fallback.

The existing workbook's older, limited Advanced Hunting mode is unchanged. Use the separate preview file for the full native implementation.

## Prerequisites

### Existing workbook

- A Microsoft Sentinel workspace with `DeviceInfo`, `Heartbeat`, `SecurityEvent`, and `Syslog` available in Log Analytics.
- Read access to the workspace and Azure resources used by the VM/DCR inventory.
- **Data source = Log Analytics (Sentinel)** for the full existing coverage view.

### Native XDR preview

> [!WARNING]
> The native XDR preview runs in the Microsoft Defender portal and requires a Microsoft Sentinel workspace connected to the unified experience. The existing Log Analytics workbook remains available for environments that ingest Defender XDR data into Sentinel.

- Defender for Endpoint access, including native `DeviceInfo` data.
- Microsoft Sentinel Reader role or higher for the selected workspace.
- Azure resource read access for the workspace GUID lookup and VM/DCR inventory.
- `Heartbeat`, `SecurityEvent`, and `Syslog` available in the connected Sentinel workspace.
- AMA deployed on the machines expected to send telemetry.

For access details, see [Advanced hunting with Microsoft Sentinel data in the Microsoft Defender portal](https://learn.microsoft.com/defender-xdr/advanced-hunting-microsoft-defender).

## Deployment and first use

### Existing workbook / fallback

If already deployed, continue using the existing workbook without changes. For a new deployment:

1. Open **Microsoft Sentinel Workbooks** and select **Add Workbook → Advanced Editor**.
2. Paste [Defender_vs_AMA.json](Defender_vs_AMA.json).
3. Select your workspace and leave **Data source = Log Analytics (Sentinel)**.
4. Confirm the results load, then save the workbook.

### Native XDR preview

1. Open **Microsoft Sentinel → Workbooks** in the [Microsoft Defender portal](https://security.microsoft.com).
2. Select **Add Workbook → Advanced Editor**.
3. Paste [Defender_vs_AMA_NativeXDR.json](Defender_vs_AMA_NativeXDR.json).
4. Select the connected Sentinel workspace using the workbook's **Workspace** filter. No workspace is prefilled.
5. Confirm the summary, coverage matrix, and merged DCR view load.
6. Save under a separate name, such as **AMA vs Defender - Native XDR Preview**; do not overwrite your existing workbook.
7. Validate the expected coverage with your own data and filters before adopting the preview in production.

Importing the preview does not replace an existing workbook or change data ingestion configuration. If it fails or its results are unexpected, continue using your existing deployment.

### Initial filters

- **Time range:** defaults to 7 days.
- **Exclude Workstations:** defaults to Yes; change it to include workstations. Mobile devices remain excluded.
- **Exclude Compliant Machines:** defaults to Yes (`MDE + AMA`). Set it to No to see compliant devices too; a fully compliant selection can otherwise produce an empty matrix.
- **AMA heartbeat seen:** select All, Yes, or No to focus on telemetry reporting.
- **OS platform:** use the text filter to narrow the results.
- **VM name:** use the DCR inventory filter to locate specific machines.

## Understanding the results

### Coverage categories

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

## Troubleshooting and limitations

| Symptom or limitation | What to check |
|-----------------------|---------------|
| Empty matrix | Check the workspace, time range, OS filter, and **Exclude Compliant Machines** setting before assuming there are no devices. |
| No heartbeat | Review the reporting window, machine power state, network connectivity, AMA service, and DCR association. Cross-check extension state in the merged view. |
| No security logs | Windows is checked through `SecurityEvent`; Linux through `Syslog`. Review the relevant DCR and log-ingestion path. |
| `WorkspaceId` parameter message in the preview | Select the top-level workspace and allow its Azure Resource Graph lookup to resolve. Check Azure read permissions if it stays unresolved. |
| Missing tables or access failures | Use a workspace connected to the Defender portal with the required Sentinel tables and permissions. These failures are not treated as zero telemetry. |
| Incomplete OS names | Some AMA versions report only `Windows` rather than a full server OS name. A server-name filter can therefore exclude such rows. |
| Native history beyond 30 days | Defender XDR normally retains 30 days of native data. Selecting a longer range does not create more history; longer history depends on retained streamed data where configured. |
| Existing workbook's limited Advanced Hunting mode | It retains the older split-table implementation. Use the separate native preview for unified coverage, or the existing Log Analytics mode as the fallback. |

## How it works / technical details

### Data and correlation

| Source | Purpose |
|--------|---------|
| `DeviceInfo` | Defender onboarding and OS information |
| `Heartbeat` | Agent reporting and last heartbeat |
| `SecurityEvent` | Windows security-log ingestion |
| `Syslog` | Linux log ingestion |
| Azure Resource Graph | VM/Arc inventory, AMA extensions, and DCR associations |

The coverage queries correlate devices using normalized short names, apply the selected filters, and return the summary and matrix. The final workbook merge combines the matrix with the ARG inventory.

In the existing workbook's full mode, all four telemetry tables are queried in Log Analytics. The native preview uses the **Advanced Hunting** data source to correlate native Defender `DeviceInfo` (`Timestamp`) with selected Sentinel workspace tables (`TimeGenerated`). ARG remains the inventory source in both versions.

### Native workspace binding and portability

The **Workspace** picker returns an Azure resource ID. A hidden, query-backed `WorkspaceId` parameter resolves its Log Analytics customer GUID through ARG and supplies it to Advanced Hunting's `selectedWorkspaceIds`. `crossComponentResources` alone does not scope Advanced Hunting queries. The coverage queries wait for the required GUID rather than silently using a default workspace.

The distributed preview has **no prefilled workspace or workspace GUID**. Defender XDR data comes from the signed-in Defender tenant; the workspace picker scopes only Sentinel data. GUIDs in `id`, `key`, and merge identifiers are internal workbook control IDs, not tenant bindings.

Portal exports can contain saved selections and cached values. Use the repository JSON as the portable template. Before sharing an edited/exported copy, remove saved workspace selections and cached GUID values, and keep parameter references rather than literal tenant-specific query scopes.

### Native query ordering

In the AMA-only branch, Sentinel telemetry joins run **before** the Defender exclusion and onboarding-status join. This preserves the coverage rules while avoiding the HTTP 500 (`ErrorCode: 9`) reproduced when a Sentinel join followed those cross-source operations. Both the matrix and summary queries use this ordering.

The standalone [Defender_AMA_coverage.kql](Defender_AMA_coverage.kql) file is unchanged by the native workbook addition.

## Related resources

- **[Sentinel Maturity Model](https://github.com/mathijsvermaat/Sentinel-Maturity)** — the tiered connector guidance model this workbook belongs to.
- **[Defender AMA Coverage walkthrough](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/procedures/defender-ama-coverage.md)** — step-by-step guide to deploying the workbook and interpreting the coverage gaps.
- **Connectors this workbook validates** — [Windows Security Events](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/connectors/windows-security-events.md), [Syslog for Linux](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/connectors/syslog-linux.md), [Windows Forwarded Events](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/connectors/windows-forwarded-events.md) and [Defender for Cloud](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/connectors/microsoft-defender-for-cloud.md).
- **[Assessment checklist](https://mathijsvermaat.github.io/sentinel-maturity-assessment.html)** — the *Defender vs AMA coverage* gap analysis records Both / AMA only / MDE only counts straight from this workbook.
