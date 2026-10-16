# Changelog

All notable changes to **New-M365PreMigrationReport.ps1** are documented here.
This project follows [Semantic Versioning](https://semver.org/) (MAJOR.MINOR.PATCH).

## [1.8.0] - 2026-10-16
### Changed
- **SharePoint-Permissions site list now comes from `Get-PnPTenantSite`,
  not the usage report.** Diagnostics added in 1.7.2 showed a live tenant
  where every one of 38 sites had a blank `Site URL` in the SharePoint
  usage report, with "conceal user, group and site names in all reports"
  confirmed off (and off long enough that propagation delay isn't the
  cause) - a Microsoft-side report quirk rather than a script bug. The
  permissions sheet now connects to the tenant's SharePoint admin center
  and lists sites via `Get-PnPTenantSite` (the admin center's own
  authoritative inventory) instead of depending on that report column, so
  it no longer silently comes up empty when that field is blank. The
  signed-in account needs Global Admin or SharePoint Admin to list sites
  this way.

## [1.7.2] - 2026-10-16
### Changed
- **Diagnostics for an empty SharePoint-Permissions sheet.** When no sites
  qualify for the permissions sheet, the script now also logs how many of
  the collected sites have a blank `SiteUrl`, how many have a `SiteUrl` that
  doesn't start with `http(s)://`, how many are marked deleted, and a sample
  of the first non-conforming `SiteUrl` value - to tell a concealed-names /
  reporting-latency issue apart from a real bug without needing to open the
  workbook first.

## [1.7.1] - 2026-10-16
### Fixed
- **SharePoint-Permissions sheet always reporting "no sites to inspect".**
  The site filter checked the usage report's `IsDeleted` field as a boolean,
  but that field is raw CSV text (`"True"`/`"False"`), and any non-empty
  string - including the literal text `"False"` - is truthy in PowerShell.
  That made every site look deleted, filtering all of them out before the
  PnP loop ran. Confirmed against a live tenant: 38 sites in the SharePoint
  usage sheet, 0 inspected for permissions. Now compares the string value
  instead of treating it as a boolean.

## [1.7.0] - 2026-10-16
### Added
- **SharePoint site permissions.** New `SharePoint-Permissions` sheet listing
  each site's built-in Owners / Members / Visitors groups and their members
  (name, login name, email, principal type). These classic site groups aren't
  exposed by Microsoft Graph, so the SharePoint workload now also connects to
  each site individually over SharePoint REST via `PnP.PowerShell` - a second,
  separate sign-in from Graph (interactive prompts once and reuses the cached
  token; app-only reuses `-ClientId`/`-CertificateThumbprint` but also needs
  the SharePoint `Sites.FullControl.All` *application* permission consented,
  since Graph app permissions don't carry over to SharePoint REST). A failed
  or declined PnP sign-in warns once and skips just that sheet rather than
  failing the run. New `-SkipSharePointPermissions` switch to skip it outright
  (e.g. tenants with a large number of sites). Summary gains counts for sites
  with permission data collected and total Owner/Member/Visitor grants.

## [1.6.0] - 2026-07-23
### Added
- **Archive mailbox detection and usage.** The Mailboxes sheet now flags
  `HasArchive`, `ArchiveStatus`, and `AutoExpandingArchive` per mailbox (from
  the existing `Get-EXOMailbox` call used for mailbox type — no extra calls).
  A new `Archive-Mailboxes` sheet collects real per-archive item count and
  size via `Get-EXOMailboxStatistics -Archive` for every archive-enabled
  mailbox. New `-SkipArchiveStats` switch to skip that sheet on large tenants
  (the Mailboxes sheet still shows `HasArchive` either way). Summary now
  counts mailboxes with an archive enabled and total archive storage.

## [1.5.1] - 2026-07-23
### Fixed
- **Usage reports (Mailbox/SharePoint/OneDrive) failing with `Forbidden
  (InvalidDownloadToken)` or "too many retries".** `Get-GraphUsageReport` used
  a raw `Invoke-MgGraphRequest -OutputFilePath` call against the reports URI.
  Those endpoints 302-redirect to a pre-signed storage URL, and
  `Invoke-MgGraphRequest` can forward the Graph bearer token onto that
  redirect, which the storage endpoint rejects — observed as an immediate
  `Forbidden (InvalidDownloadToken)` for SharePoint/OneDrive, and as a multi-
  minute retry loop ending in "Too many retries performed" for Mailbox usage.
  Switched to the typed `Get-MgReportMailboxUsageDetail` /
  `Get-MgReportOneDriveUsageAccountDetail` / `Get-MgReportSharePointSiteUsageDetail`
  cmdlets (`Microsoft.Graph.Reports` module, now auto-installed when the
  Exchange or SharePoint workload runs), which handle the redirect correctly.

## [1.5.0] - 2026-07-23
### Added
- **Log file.** The full console output of the run is now also written to a
  timestamped `New-M365PreMigrationReport_<yyyy-MM-dd_HHmmss>.log` file in the
  same folder as the script (via `Start-Transcript`), separate from the
  `.xlsx` report. New `-SkipLogFile` switch to disable it. If the transcript
  can't be started, the script warns and continues without a log file instead
  of aborting the run; the self-update exit path stops the transcript cleanly
  before restarting.

## [1.4.0] - 2026-07-23
### Changed
- **Tenant sheet: domains split by type.** The single `VerifiedDomains` column
  is now two columns, `OnMicrosoftDomains` and `VanityDomains` — the latter is
  the list that actually needs re-verifying and re-pointing on the target
  tenant during a tenant-to-tenant cutover, so it's no longer buried in a mixed
  list with the tenant-issued `.onmicrosoft.com` domains. The MX / inbound mail
  routing lookup now reads `VanityDomains` directly instead of filtering out
  `.onmicrosoft.com` from the old combined list.

## [1.3.1] - 2026-07-22
### Fixed
- **Public folder `FolderPath` was unusable.** Exchange Online's V3 module returns
  `Get-PublicFolder`/`Get-PublicFolderStatistics`'s `FolderPath` as an array of path
  segments, not a single string like the classic module. Used as-is, this made the
  `PublicFolders` sheet's `FolderPath` column print the literal type name
  (`System.Collections.ArrayList`) and silently broke the stats/mail-enabled
  lookups (an ArrayList has no value-based equality, so the dictionary keyed by it
  never matched), leaving `ItemCount`, `SizeGB`, `LastModified`,
  `PrimarySmtpAddress`, and `EmailAddresses` blank for every row. Added
  `Get-PfPathString` to normalize the value to `\Segment1\Segment2` before it's
  used as a column or a lookup key.

## [1.3.0] - 2026-07-22
### Added
- **Public folder discovery.** New `PublicFolders` sheet inventories the public
  folder tree via Exchange Online (`Get-PublicFolder -Recurse`), with per-folder
  item count/size (`Get-PublicFolderStatistics`) and mail-enabled SMTP/proxy
  addresses (`Get-MailPublicFolder`) — public folders have no native
  tenant-to-tenant migration path, so this is the inventory a migration team
  plans content migration from. A new `PublicFolder-Permissions` sheet captures
  per-folder client permissions (Owner/Editor/Reviewer/etc.), which also don't
  transfer and must be re-applied on the target. New switches:
  `-SkipPublicFolders` (skips both sheets) and `-SkipPublicFolderPermissions`
  (keeps the folder inventory, skips the slower one-call-per-folder permissions
  sheet). Public folder discovery runs even with `-SkipMailboxType`, since it
  only needs the Exchange Online connection, not the mailbox list. The Summary
  tab notes whether public folders are enabled for the tenant.

## [1.2.0] - 2026-07-17
### Added
- **Delegation `AutoMapping` column.** The Delegation sheet now flags automapping
  for each grant. Exchange Online doesn't expose the AutoMapping flag for existing
  FullAccess permissions (it's held in `msExchDelegateListLink`, which the mailbox
  cmdlets don't return), so the value reflects Microsoft's documented default:
  individual FullAccess grants show `On (default)`; SendAs/SendOnBehalf show `N/A`.
  A `-AutoMapping $false` grant can't be distinguished via PowerShell. The Summary
  now also counts FullAccess grants.

## [1.1.0] - 2026-07-17
### Added
- **Self-update.** On start the script checks GitHub for a newer version — reading
  the latest published **Release** (falling back to the `main` branch if none),
  prints that release's notes, and — after you confirm — downloads and replaces
  itself (with a `.bak` backup and syntax validation of the download). New
  switches: `-SkipUpdateCheck`, `-AutoUpdate`, `-UpdateToken`.
- **Portable output location.** Reports now default to
  `Documents\M365 Pre-Migration Reports\`, resolved per-user at runtime and
  OneDrive-redirect aware, so the script runs unmodified on any machine.

## [1.0.0] - 2026-07-14
### Added
- Initial release. Inventories a source Microsoft 365 tenant for a
  tenant-to-tenant migration into a multi-tab Excel workbook:
  - **Identity:** tenant overview, license SKUs (friendly names via Microsoft's
    published catalog), users, MFA/auth-method registration, domains
    (managed vs federated, with federation issuer URIs).
  - **Exchange:** mailbox usage + type (User/Shared/Room/Equipment via Exchange
    Online), per-mailbox delegation (FullAccess/SendAs/SendOnBehalf), mail-enabled
    groups, and MX / inbound-mail-filtering lookup.
  - **SharePoint & OneDrive:** site and per-user storage usage.
  - **Teams & Groups:** M365 groups and Teams with owner/member counts.
