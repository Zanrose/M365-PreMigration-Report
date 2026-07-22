# Changelog

All notable changes to **New-M365PreMigrationReport.ps1** are documented here.
This project follows [Semantic Versioning](https://semver.org/) (MAJOR.MINOR.PATCH).

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
