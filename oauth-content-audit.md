# OAuth Consent Screen — Content Audit Report

**Project:** E.M.P.O.W.E.R (DSWD Point of Sale System)
**Package name:** `dswd_slp` (`com.example.dswd_slp`)
**Audit date:** July 23, 2026
**Auditor:** Automated codebase analysis

---

## 1. Files Inspected

The following source code files were examined to verify all claims made in the public-facing pages (`index.html`, `privacy.html`, `terms.html`):

### Services (data handling & networking)

| File | Lines | Relevance |
|------|-------|-----------|
| `lib/services/google_drive_auth_service.dart` | 156 | **OAuth implementation**, scope definition, token handling |
| `lib/services/google_drive_service.dart` | 368 | All Drive API operations (create, list, download, delete) |
| `lib/services/backup_service.dart` | 258 | Backup orchestration, auth verification |
| `lib/services/backup_archive_service.dart` | 356 | Backup ZIP creation, checksum, validation |
| `lib/services/restore_service.dart` | 318 | Restore orchestration, safety rollback |
| `lib/services/restore_safety_service.dart` | 30 | Local safety backup management |
| `lib/services/db_service.dart` | 1566 | SQLite schema, `clearData()`, `onUpgrade` |
| `lib/services/notification_service.dart` | 91 | EmailJS integration (product alerts) |
| `lib/services/auto_sync_service.dart` | 24 | Stubbed sync (no real API) |
| `lib/services/session_service.dart` | 25 | Session clear, SharedPreferences keys |
| `lib/services/user_media_service.dart` | 46 | Image storage paths |
| `lib/services/audit_log_service.dart` | 107 | Audit trail implementation |

### Data access (repositories)

| File | Lines | Relevance |
|------|-------|-----------|
| `lib/repositories/backup_repository.dart` | 292 | Backup settings CRUD, config fields |
| `lib/repositories/product_repository.dart` | — | Product data handling |

### Models

| File | Lines | Relevance |
|------|-------|-----------|
| `lib/models/backup_settings.dart` | 81 | Settings schema (stored fields) |
| `lib/models/backup_metadata.dart` | 146 | Backup metadata structure |

### Entry & configuration

| File | Lines | Relevance |
|------|-------|-----------|
| `lib/main.dart` | 70 | App bootstrap, FFI init, scheduler init |
| `lib/app_router.dart` | 415 | Navigation, feature listing |
| `pubspec.yaml` | 71 | Dependencies, version, package name |
| `android/app/src/main/AndroidManifest.xml` | 51 | Permissions, app label |

### Design documentation

| File | Lines | Relevance |
|------|-------|-----------|
| `google-drive-backup-shared-account-updated.md` | 388 | Backup design spec, architecture decisions |
| `README.md` | 135 | App overview, features, team |
| `AGENTS.md` (Inventory/) | — | Architecture guidelines |
| `AGENTS.md` (root/) | — | Repo-level instructions |

### UI screens (feature validation)

| File | Lines | Relevance |
|------|-------|-----------|
| `lib/views/screens/about_app_screen.dart` | 863 | Developer identity, organizational claims |
| `lib/views/screens/login_screen.dart` | — | PIN auth UI |

---

## 2. Database Tables Relevant to User Data

### Tables storing personal or operational data

| Table | Personal Data | Operational Data |
|-------|--------------|------------------|
| `account` | SLPA name, full name, mobile, PIN, security Q&A, profile image | Organization UUID |
| `slpa_member` | Full name, mobile (UNIQUE), PIN, security Q&A | Account FK |
| `customer` | Full name, phone, municipality, barangay, landmark | Credit limit, available credit |
| `product` | — | Name, category, prices, quantity, image |
| `stock_in` | — | Quantity, purchase price, image |
| `sales` | Created-by member name | Total, sale type |
| `expenses` | Created-by member name | Amount, category, receipt image |
| `audit_log` | Member name, module, action | Old/new values (JSON) |
| `backup_settings` | **Google Account ID**, **Google Account email** | Folder IDs, timestamps, operation state |
| `member_duty_log` | Member name | Shift start/end times |
| `customer_payment` | — | Amount, timestamp |
| `payable` | Supplier name | Amounts, due dates |

### Tables with `sync_status` / `is_deleted` columns (future-ready)

`product`, `stock_in`, `inventory_lot`, `lot_transformation`, `sales`, `customer`,
`expenses`, `payable`, `capital_management`, `slpa_member`, `member_duty_log`

---

## 3. Google OAuth Implementation

### Implementation file
`lib/services/google_drive_auth_service.dart` (lines 1–156)

### Scope (line 25)
```dart
static const driveScope = 'https://www.googleapis.com/auth/drive.file';
```

### Server Client ID (lines 26–30)
```dart
static const _serverClientId = String.fromEnvironment(
  'GOOGLE_SERVER_CLIENT_ID',
  defaultValue:
      '941990864471-qd672ljttaoqufvb4or3mvkd2hbnunfa.apps.googleusercontent.com',
);
```

**Note:** The Google Cloud OAuth client ID is **hardcoded as a fallback**. It should be provided at build time via `--dart-define=GOOGLE_SERVER_CLIENT_ID=...`. The hardcoded default is exposed in the source code — this should be treated as a development/testing credential and replaced for production builds.

### Library
`google_sign_in: ^7.2.0`

### Authentication methods used

| Method | Purpose |
|--------|---------|
| `GoogleSignIn.instance.initialize()` | Initialize with serverClientId |
| `GoogleSignIn.instance.authenticate()` | Interactive sign-in (with `scopeHint: [driveScope]`) |
| `GoogleSignIn.instance.attemptLightweightAuthentication()` | Silent token refresh |
| `GoogleSignIn.instance.signOut()` | Sign out current session |
| `GoogleSignIn.instance.disconnect()` | Revoke app access (remove all tokens) |

### Token storage
- OAuth access and refresh tokens are stored by the `google_sign_in` plugin in **Android Keystore** (secure hardware-backed storage).
- The App **does not** store tokens in SQLite or in backup archives.

---

## 4. OAuth Scopes Found

| Scope | Source File | Line | Purpose | Verification |
|-------|-------------|------|---------|-------------|
| `https://www.googleapis.com/auth/drive.file` | `google_drive_auth_service.dart` | 25 | Create, read, update, delete files created by the App in Google Drive | **Confirmed** |

No other OAuth scopes are declared anywhere in the codebase.

---

## 5. Google Drive API Operations Found

All in `lib/services/google_drive_service.dart` using `googleapis/drive/v3.dart` (package: `googleapis: ^16.0.0`).

| Operation | API Method | Line(s) | Description |
|-----------|-----------|---------|-------------|
| List files/folders | `api.files.list()` | 34, 59–65, 112–127, 214–220, 323–335 | Query Drive for EMPOWER root folder, organization folders, and backup files |
| Create folder | `api.files.create()` | 41–47, 69–80 | Create EMPOWER root and organization subfolders with `appProperties` |
| Upload backup | `api.files.create()` with `uploadMedia` | 161–183 | Upload ZIP backup archive with metadata in `appProperties` |
| Update folder name | `api.files.update()` | 83–88 | Rename organization folder when SLPA name changes |
| Download backup | `api.files.get()` with `downloadOptions: DownloadOptions.fullMedia` | 245–248 | Download backup ZIP for restore |
| Delete old backup | `api.files.delete()` | 294 | Delete backups exceeding retention limit (10) |

### Drive folder structure
```
Google Drive /
└── EMPOWER/                          (appProperty: slp_kind=root)
    └── [Organization Name]/           (appProperty: slp_kind=organization, organization_id=<UUID>)
        ├── backup_2026-07-21_23-30-15.zip
        └── backup_2026-07-22_23-30-15.zip
```

### Backup archive contents
```
backup_YYYY-MM-DD_HH-mm-ss.zip
├── database.db          (SQLite VACUUM INTO snapshot)
├── images/              (user-uploaded images from: account, product, stock_in, expenses tables)
│   ├── account_1_profile_image.jpg
│   └── ...
└── metadata.json        (JSON: formatVersion, organizationId, organizationName,
                          appVersion, buildNumber, databaseVersion, backupDate,
                          backupType, googleAccountId, databaseSha256, media, warnings)
```

### Backup retention
- Latest 10 backups kept per organization (`enforceRetention`, default `keep = 10`).
- Older backups deleted automatically after successful upload.

---

## 6. Personal or Organizational Data Handled

### Data stored locally (device SQLite + filesystem)

| Data | Table(s) | Notes |
|------|----------|-------|
| Full name (first, middle, last) | `account`, `slpa_member`, `customer` | Multiple members + customers |
| Mobile number | `account` (has `server_id`/`sync_status`), `slpa_member` (UNIQUE), `customer` (UNIQUE) | Unique per member and customer |
| 4-digit PIN | `account.pin`, `slpa_member.pin` | Stored as **plain text** |
| Security question & answer | `account.security_question_id`, `account.security_answer`, `slpa_member.security_question_id`, `slpa_member.security_answer` | PIN recovery (Cebuano questions) |
| Profile image | `account.profile_image` (file path) | Copied to `user_images/` |
| Product images | `product.image`, `stock_in.image` | Copied to `user_images/` |
| Expense receipts | `expenses.receipt` | Copied to `user_images/` |
| Customer address | `customer.municipality`, `customer.barangay`, `customer.landmark` | Region 7 PSGC codes |
| Transaction history | `transaction_history`, `sales`, `sale_item`, `expenses`, etc. | Full operational data |
| Audit log | `audit_log` | All CUD actions with actor |
| Session data | SharedPreferences | `memberId`, `accountId`, `mobileNumber`, names, `slpaName` |
| Backup config | `backup_settings` | Google Account ID, email, folder IDs, timestamps |

### Data uploaded to Google Drive (backup archives)

| Data | How | Notes |
|------|-----|-------|
| Full SQLite database | `database.db` inside ZIP | Contains ALL local data including PINs (plaintext), member info, customers, transactions |
| User-uploaded images | `images/` directory inside ZIP | Account, product, stock-in images, expense receipts |
| Metadata | `metadata.json` | Organization identity, app version, Google Account ID, checksums |

### Data NOT stored or collected

| Data Type | Verified Absent | Evidence |
|-----------|----------------|----------|
| Advertising ID | No `com.google.android.gms.ads` dependency | pubspec.yaml |
| GPS location | No location packages or permissions | pubspec.yaml, AndroidManifest.xml |
| Device identifiers | No telephony or device ID packages | pubspec.yaml |
| Contact list | No contacts packages | pubspec.yaml |
| SMS / call logs | No SMS or phone packages | pubspec.yaml |
| Analytics events | No Firebase Analytics, Google Analytics, Amplitude, Mixpanel | pubspec.yaml |
| Crash reports | No Firebase Crashlytics, Sentry | pubspec.yaml |
| Browsing history | Not applicable (mobile app) | — |

---

## 7. Statement Verification Matrix

Each statement in the public pages is mapped to its source code evidence:

| Public Page Statement | Verified? | Source |
|-----------------------|-----------|--------|
| "Offline-first mobile application" | **Confirmed** | All data in local SQLite; auto-sync is stubbed (500ms delay) |
| "4-digit PIN authentication" | **Confirmed** | `login_view_model.dart`, `slpa_member.pin` column |
| "PIN stored as plain text" | **Confirmed** | `slpa_member.pin TEXT`, `account.pin TEXT` — no hash function found |
| "Google Drive backup is optional" | **Confirmed** | `backup_service.dart` requires explicit configuration |
| "OAuth scope: drive.file" | **Confirmed** | `google_drive_auth_service.dart:25` |
| "Tokens stored in Android Keystore, not SQLite" | **Confirmed** | No token columns in any table; design doc confirms Keystore |
| "Backup archives are not encrypted" | **Confirmed** | Design doc: "ZIP backups are not encrypted" |
| "Backup contains full database snapshot" | **Confirmed** | `backup_archive_service.dart:79`: `VACUUM INTO` database file |
| "10 latest backups retained" | **Confirmed** | `google_drive_service.dart:303`: `keep = 10` |
| "EmailJS sends product edit/delete alerts" | **Confirmed** | `notification_service.dart:16-52` |
| "Developed by CTU Ginatilan students" | **Confirmed** | `about_app_screen.dart`: CTU branding, student team list |
| "App works without internet" | **Confirmed** | All features use local SQLite; only backup/notifications need network |
| "No analytics or tracking SDKs" | **Confirmed** | pubspec.yaml: no analytics packages |
| "Clear All Data removes 23 tables" | **Confirmed** | `db_service.dart:1513-1536`: 23 tables listed in `clearData()` |
| "Disconnect does not delete existing Drive backups" | **Confirmed** | `backup_service.dart:252-257`: `disconnectGoogleDrive` clears config only |
| "Revoke via Google Account permissions" | **Confirmed** | `GoogleSignIn.instance.disconnect()` in `google_drive_auth_service.dart:136-139` |
| "Official DSWD product" | **NOT confirmed** | No endorsement letter or authorization document in repo |
| "Official CTU product" | **NOT confirmed** | CTU logo and name appear in About screen, but no official endorsement document |
| "Deliberately designed for elderly users" | **NOT included** | Zoom feature exists but not claimed in public pages |

---

## 8. Unverifiable Statements (Not Included in Public Pages)

The following were intentionally **excluded** from the public pages because source code could not verify them:

- Any claim of official DSWD endorsement or authorization
- Any claim of official CTU endorsement or authorization
- Any guarantee of data security or encryption
- Any specific data retention period (no retention policy in code)
- Any compliance certification (e.g., GDPR, DP Act)
- Any statement about frequency or reliability of automatic backups (Android may delay)

---

## 9. Placeholders / Manual Inputs Required

| Item | Status |
|------|--------|
| `[DEVELOPER NAME]` | **Filled** — "Al Duane Cuevas Mirasol and Kevin Rey Mejares" |
| `[SUPPORT EMAIL]` | **Filled** — dswdslpempower@gmail.com |
| `[EFFECTIVE DATE]` | **Filled** — July 23, 2026 |
| `[WEBSITE DOMAIN]` | **Filled** — dswd-slp-empower.github.io/empower |
| `[COUNTRY]` | **Filled** — Republic of the Philippines |
| Google Cloud OAuth consent screen configuration | **Needs manual setup** in Google Cloud Console |
| Google Cloud Project ID / OAuth client ID | **Needs production value** — current default is hardcoded dev/testing ID |
| EmailJS account ownership | **Needs confirmation** — hardcoded credentials (`service_mu1vij8`, `template_bw3oaa8`, `p4eMgeo7OeM3hAybI`) in source |
| GitHub Pages deployment | **Needs manual action** — enable GitHub Pages on the repository |

---

## 10. OAuth Verification Risks and Inconsistencies

### Risk 1: Hardcoded OAuth Client ID
- **File:** `lib/services/google_drive_auth_service.dart:26-30`
- **Details:** A default server client ID is hardcoded. Google recommends providing this via secure build-time configuration only. The default value is publicly visible in the source code.
- **Recommendation:** Configure the real OAuth client ID via `--dart-define=GOOGLE_SERVER_CLIENT_ID=...` and remove or replace the hardcoded default for production builds.

### Risk 2: Plaintext PIN Storage
- **File:** `lib/services/db_service.dart` — `account.pin` and `slpa_member.pin` are TEXT columns
- **Details:** PINs are stored without hashing. Since backups include the full database, PINs are present in Drive backup archives.
- **Impact on OAuth review:** Google may flag this as a security concern, though it does not directly involve Google user data.
- **Recommendation:** Hash PINs before storage. Document the limitation in privacy policy (already done).

### Risk 3: Cleartext Traffic Permitted
- **File:** `android/app/src/main/AndroidManifest.xml:11`
- **Details:** `android:usesCleartextTraffic="true"` allows unencrypted HTTP connections.
- **Impact on OAuth review:** May raise security concerns during review. EmailJS calls use standard HTTP.
- **Recommendation:** Remove or limit to debug builds.

### Risk 4: Hardcoded EmailJS Credentials
- **File:** `lib/services/notification_service.dart:10-13`
- **Details:** EmailJS service ID, template ID, and public API key are hardcoded and publicly visible.
- **Impact on OAuth review:** Not directly related to Google data, but indicates general security posture.
- **Recommendation:** Move to build-time configuration or a backend proxy.

### Risk 5: Single-Device Shared Access
- **Design doc:** "One device per organization"
- **Details:** All members share one device. There is no per-user data isolation at the OS level.
- **Impact on OAuth review:** The shared Google Account model means multiple users authenticate with the same Google Account for backup. This should be clearly documented.

### Risk 6: Stubbed Sync with Premature DB Columns
- **File:** `lib/services/auto_sync_service.dart` (stubbed)
- **Details:** Many tables have `server_id`, `sync_status`, `last_synced_at` columns prepared for remote sync, but the sync service is a placeholder (500ms delay + debugPrint). If a future sync implementation is added, OAuth scope requirements may change.
- **Recommendation:** Document that sync is not implemented. If adding sync later, consider if `drive.file` scope remains sufficient.

### Risk 7: Scope Sufficiency Review
- **Current scope:** `drive.file` — restricts access to files created by the App.
- **Sufficiency:** Sufficient for all current Drive operations (create, list, download, delete).
- **No additional scopes needed:** The App does not read user metadata, other Drive files, or perform admin operations.

---

## 11. Summary of Created Files

| File | Purpose |
|------|---------|
| `docs/index.html` | Public home page with app description, feature list, and navigation to privacy/terms |
| `docs/privacy.html` | Full Privacy Policy covering data collection, Google API Limited Use compliance, storage, retention, deletion, third-party services |
| `docs/terms.html` | Terms of Service covering eligibility, acceptable use, disclaimers, liability limits, governing law (Philippines) |
| `docs/styles.css` | Responsive CSS for GitHub Pages deployment with mobile-friendly layout |
| `docs/oauth-content-audit.md` | This audit document — maps every public statement to source code evidence |

### Navigation structure
```
index.html  ←→  privacy.html
   ↓              ↓
terms.html  ←→  (linked in footer)
```

### Deployment instructions
1. Push `docs/` directory to the GitHub repository
2. Go to repository Settings → Pages
3. Set source to "Deploy from a branch" → branch `main` → folder `/docs`
4. Pages will be available at `https://dswd-slp-empower.github.io/empower/`

---

*This audit was produced by automated codebase analysis. All source-referenced claims are accurate as of July 23, 2026. Statements requiring legal, business, or organizational verification are marked accordingly.*
