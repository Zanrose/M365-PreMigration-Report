# Changelog

All notable changes to **New-M365PreMigrationReport.ps1** are documented here.
This project follows [Semantic Versioning](https://semver.org/) (MAJOR.MINOR.PATCH).

## [1.1.0] - 2026-07-17
### Added
- **Self-update.** On start the script checks GitHub for a newer version, prints
  the changelog entries newer than the installed version, and — after you confirm
  — downloads and replaces itself (with a `.bak` backup and syntax validation of
  the download). New switches: `-SkipUpdateCheck`, `-AutoUpdate`, `-UpdateToken`.
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
