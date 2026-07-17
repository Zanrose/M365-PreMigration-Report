# M365 Pre-Migration Report

`New-M365PreMigrationReport.ps1` inventories a **source** Microsoft 365 tenant for a
**tenant-to-tenant** migration and produces a multi-tab Excel workbook.

## What it collects

| Worksheet         | Workload            | Source |
|-------------------|---------------------|--------|
| `Summary`         | headline roll-up    | all of the below |
| `Tenant`          | always              | `/organization` |
| `Licenses`        | Identity            | `/subscribedSkus` |
| `Users`           | Identity            | `/users` (incl. per-user licenses, dir-sync, guest/member) |
| `MFA-Registration`| Identity            | `/reports/authenticationMethods/userRegistrationDetails` |
| `Domains`         | Identity            | `/domains` (+ `federationConfiguration`) — Managed vs Federated |
| `Mailboxes`       | Exchange            | `getMailboxUsageDetail` report + Exchange Online (`Get-EXOMailbox`) for mailbox type |
| `Delegation`      | Exchange            | Exchange Online — FullAccess / SendAs / SendOnBehalf per mailbox |
| `MailGroups`      | Exchange            | `/groups` (DLs / mail-enabled / M365) |
| `MX-Records`      | Exchange            | DNS MX lookup per verified domain (inbound mail routing / 3rd-party filter) |
| `SharePoint`      | SharePoint          | `getSharePointSiteUsageDetail` report |
| `OneDrive`        | SharePoint          | `getOneDriveUsageAccountDetail` report |
| `Teams-Groups`    | Teams               | `/groups` (Unified) + owner/member counts |

## Requirements

- Windows PowerShell 5.1 or PowerShell 7+
- Modules (auto-installed to CurrentUser if missing): `Microsoft.Graph.Authentication`,
  `ImportExcel`, and `ExchangeOnlineManagement` (only when the Exchange workload runs
  without `-SkipMailboxType`)
- Graph permissions: `Organization.Read.All`, `User.Read.All`, `Group.Read.All`,
  `GroupMember.Read.All`, `Directory.Read.All`, `Sites.Read.All`, `Reports.Read.All`,
  `AuditLog.Read.All`, `Domain.Read.All`

## Usage

By default the report is written to **`Documents\M365 Pre-Migration Reports\`** and named
`<TenantName>-PreMigration-<date>.xlsx`. The Documents location is resolved per-user at
runtime (and follows OneDrive's redirected Documents folder if Known Folder Move is enabled),
so the script is portable across machines and users — no paths to edit. Override with
`-OutputPath <folder>` or `-OutputPath <full-path.xlsx>`.

```powershell
# Interactive, all workloads -> Documents\M365 Pre-Migration Reports\<Tenant>-PreMigration-<date>.xlsx
.\New-M365PreMigrationReport.ps1

# Pick workloads + 90-day usage window, custom output folder
.\New-M365PreMigrationReport.ps1 -OutputPath C:\Reports -Workload Identity,Exchange -UsagePeriod D90

# Unattended (app registration with a certificate)
.\New-M365PreMigrationReport.ps1 -UseAppOnly -TenantId contoso.onmicrosoft.com `
    -ClientId <app-guid> -CertificateThumbprint <thumbprint> -OutputPath C:\Reports
```

Parameters: `-OutputPath`, `-TenantId`, `-Workload` (Identity/Exchange/SharePoint/Teams),
`-UsagePeriod` (D7/D30/D90/D180), `-UseAppOnly` / `-ClientId` / `-CertificateThumbprint`,
`-SkipModuleInstall`, `-SkipMailboxType`, `-SkipDelegation`, `-OfflineSkuNames`,
`-SkuCatalogPath`, `-SkuCatalogUrl`, `-SkipUpdateCheck`, `-AutoUpdate`, `-UpdateToken`.

## Self-update

On start the script compares its `$ScriptVersion` against the copy in this GitHub repo
(`main` branch). If a newer version exists it prints the relevant [CHANGELOG](CHANGELOG.md)
entries and asks before downloading and replacing itself. The current file is backed up to
`<script>.v<old>.bak` first, and the download is syntax-checked before it's written, so a bad
download can't clobber a working script. After an update it exits and asks you to re-run.

- `-SkipUpdateCheck` — never check (e.g. offline / locked-down environments).
- `-AutoUpdate` — apply a newer version without prompting (for scheduled/unattended runs).
- `-UpdateToken <PAT>` — a GitHub token, **only needed while the repo is private**. If the
  GitHub CLI (`gh`) is installed and signed in, the script uses that automatically and no
  token is needed. Once the repo is public, the check works anonymously.

## License friendly names

Graph only returns a license's `skuId` (GUID) and `skuPartNumber` (e.g. `SPE_E5`) —
never the marketing name ("Microsoft 365 E5"). To get full names the script downloads
Microsoft's published **"Product names and service plan identifiers for licensing"** CSV
(~620 SKUs) at runtime and resolves each license: catalog by GUID → catalog by part
number → built-in static map → raw part number. No tenant permissions are involved; it's
a public download.

- `-OfflineSkuNames` — skip the download and use only the small built-in map (air-gapped /
  no internet). Unmapped SKUs show their `skuPartNumber`.
- `-SkuCatalogPath <file.csv>` — use a previously saved copy of the CSV instead of downloading.
- `-SkuCatalogUrl <url>` — override the download URL if Microsoft moves the file.

If the download fails for any reason the script logs a warning and automatically falls back
to the built-in map — it never aborts the run over license names.

## Important notes / known limitations

- **Concealed report names.** If *Admin center > Settings > Org settings > Reports >
  "Display concealed user, group and site names"* is **ON**, the usage reports return
  blank/GUID names instead of UPNs and site URLs. Turn it off before running for full detail.
- **Mailbox types.** The Graph usage report can't tell a Shared/Room/Equipment mailbox from a
  user mailbox, so the Exchange workload also connects to **Exchange Online** (`Get-EXOMailbox`)
  and fills in `MailboxType` + `IsShared` on the Mailboxes sheet (and a Shared/Room/Equipment
  count on the Summary). This adds the `ExchangeOnlineManagement` module and a second sign-in.
  Pass `-SkipMailboxType` to skip it and stay pure-Graph — `MailboxType` then reads
  "Unknown (skipped)". If the ExO connection fails, the run continues and the type reads
  "Unknown".
- **Delegation.** FullAccess / SendAs / SendOnBehalf grants are inventoried per mailbox
  (explicit grants only — inherited rights and `NT AUTHORITY\SELF` are excluded). These
  permissions do **not** transfer in a tenant-to-tenant move and must be re-applied on the
  target. Skipped with `-SkipDelegation`; needs the same ExO connection as mailbox type.
- **MX / inbound filtering.** Each verified (non-`onmicrosoft.com`) domain is MX-resolved.
  If the lowest-preference host isn't `*.mail.protection.outlook.com`, `ThirdPartyInbound`
  is `True` and `Provider` names the likely gateway (Mimecast, Proofpoint, Barracuda, Cisco
  IronPort, Google, etc.). A third-party filter must be re-pointed at the target tenant's EOP
  at cutover. Requires outbound DNS from the machine running the script.
- **Federated domains.** The `Domains` sheet reports each domain's `AuthenticationType`
  (`Managed` vs `Federated`) with the federation issuer / passive sign-in URI for federated
  ones. Federation (ADFS or a third-party IdP) does **not** transfer in a tenant-to-tenant
  move — federated domains must be converted to managed or re-federated against the target
  tenant, so they're flagged on the Summary too.
- **Usage data lag.** Microsoft usage reports are typically 1–2 days behind real time.
- One failing workload won't abort the run — it logs an error and writes an empty sheet.

## Suggested next steps for a tenant-to-tenant cutover

- Add a destination-tenant capacity check (compare source storage totals vs. target license entitlements).
- Add Conditional Access policy export (`/identity/conditionalAccess/policies`) — these don't migrate and must be rebuilt.
- Add SPF/DKIM/DMARC records to the MX pass — the other mail-authentication config to rebuild on the target.
- Add directory-role assignments (who holds Global Admin, Exchange Admin, etc.) — rebuilt manually on the target.

## License

MIT — see [LICENSE](LICENSE). Free to use, modify, and redistribute with attribution.
