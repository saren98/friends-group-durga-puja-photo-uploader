# Friends Group Durga Puja Photo Uploader

A PHP-based photo uploader and gallery for Friends Group Durga Puja, hosted on InfinityFree. The project supports public gallery access, photo uploads to Google Drive, an admin area, and visitor analytics.

> **Note:** This documentation describes the features and structure confirmed during development. Adjust filenames or behavior in this README if the deployed project changes.

## Features

- Public photo gallery.
- Photo upload workflow integrated with Google Drive.
- Admin login and protected admin pages.
- Admin session timeout and session cookie protections.
- Visitor analytics for public gallery page requests.
- Visitor history with device, operating system, browser, IP address, first/last visit, and visit count.
- Daily, weekly, and monthly traffic chart views with date filtering.
- CSV export of visitor analytics.
- Admin activity logging.
- Prepared SQL statements through PDO.

## Technology Stack

- PHP 8.2
- MySQL / MariaDB
- PDO
- HTML, CSS, JavaScript
- Google Drive API integration
- InfinityFree hosting and phpMyAdmin

## Project Structure

The following is a high-level view of the relevant files. The exact deployed structure may include additional files.

```text
htdocs/
├── index.php
├── admin_config.php              # Private database/admin configuration
├── config.php                    # Google Drive/uploader configuration
├── drive.php                     # Google Drive integration
├── includes/
│   └── visitor-tracker.php        # Public visitor analytics tracker
├── public/
│   ├── index.php                  # Public gallery
│   ├── upload.php                 # Upload endpoint
│   └── assets/
│       ├── style.css
│       └── app.js
└── admin/
    ├── dashboard.php
    ├── analytics.php
    ├── export-analytics.php
    └── includes/
        ├── bootstrap.php
        ├── layout.php
        └── footer.php
```

## How Visitor Analytics Works

The tracker is included at the beginning of the public gallery page, before HTML output:

```php
<?php
require_once dirname(__DIR__) . '/includes/visitor-tracker.php';
?>
<!doctype html>
```

The tracker:

1. Runs for public `GET` requests and skips command-line execution.
2. Creates or reuses a first-party `puja_visitor_id` cookie.
3. Uses a pseudonymous random visitor token to distinguish browsers.
4. Estimates device type, operating system, and browser from the User-Agent string.
5. Records the visitor's IP address and a SHA-256 IP hash.
6. Updates `visitor_analytics` and inserts a pageview into `visitor_pageviews`.
7. Uses a database transaction and catches tracking errors so analytics failures do not break the public gallery.

Visitor counts are cookie-based estimates. Clearing cookies or using a different browser/device can create another visitor record. Shared IP addresses, VPNs, and proxies can also affect interpretation.

### Time zone

The tracker sets PHP's default time zone to `Asia/Kolkata` and sets the MySQL session time zone to `+05:30`. The analytics dashboard should use consistent local-time handling when querying and displaying timestamps.

## Database Tables

### `visitor_analytics`

Stores one row per visitor token.

| Column | Purpose |
|---|---|
| `id` | Auto-incrementing row ID |
| `visitor_token` | Unique pseudonymous visitor identifier |
| `first_visit` | First recorded visit timestamp |
| `last_visit` | Most recent recorded visit timestamp |
| `visit_count` | Number of tracked requests for this token |
| `last_ip_hash` | SHA-256 hash of the IP address |
| `ip_address` | Recorded IP address |
| `device_type` | Estimated device category |
| `operating_system` | Estimated operating system |
| `browser` | Estimated browser |

### `visitor_pageviews`

Stores individual pageview events.

| Column | Purpose |
|---|---|
| `id` | Auto-incrementing row ID |
| `visitor_token` | Visitor token associated with the event |
| `page_path` | Requested URL path |
| `visited_at` | Pageview timestamp |

### SQL schema

Run the following SQL in phpMyAdmin only if these tables do not already exist. If the tables already exist, do not recreate them; compare the existing schema first.

```sql
CREATE TABLE IF NOT EXISTS `visitor_analytics` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `visitor_token` CHAR(64) NOT NULL,
    `first_visit` DATETIME NOT NULL,
    `last_visit` DATETIME NOT NULL,
    `visit_count` INT UNSIGNED NOT NULL DEFAULT 1,
    `last_ip_hash` CHAR(64) DEFAULT NULL,
    `device_type` VARCHAR(20) DEFAULT NULL,
    `operating_system` VARCHAR(50) DEFAULT NULL,
    `browser` VARCHAR(50) DEFAULT NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `unique_visitor_token` (`visitor_token`),
    KEY `idx_last_visit` (`last_visit`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `visitor_pageviews` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `visitor_token` CHAR(64) NOT NULL,
    `page_path` VARCHAR(255) NOT NULL,
    `visited_at` DATETIME NOT NULL,
    PRIMARY KEY (`id`),
    KEY `idx_visited_at` (`visited_at`),
    KEY `idx_visitor_visited` (`visitor_token`, `visited_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

If `visitor_analytics` already exists and does not have the `ip_address` column, the following migration adds it:

```sql
ALTER TABLE `visitor_analytics`
ADD COLUMN `ip_address` VARCHAR(45) DEFAULT NULL AFTER `last_ip_hash`;
```

Run this `ALTER TABLE` only if the column is absent; otherwise MySQL will report a duplicate-column error.

## Admin Analytics

The admin analytics page provides:

- Total recorded visitor tokens.
- A recent-activity estimate for the last five minutes.
- Today's visitor and pageview statistics.
- Device, operating system, and browser summaries.
- Visitor history search and pagination.
- Daily, weekly, and monthly traffic views.
- Start and end date filters.
- CSV download of visitor history.

**Interpretation notes**

- “Live visitors” is an estimate based on recent tracked activity, not a guaranteed count of people currently viewing the page.
- Visitor identities are based on browser cookies, not verified human identities.
- A “returning visitor” figure calculated from total visitors and today's new visitors is an estimate, not a complete historical cohort analysis.
- The traffic chart counts pageview events; it does not necessarily represent unique people.

## Configuration and Deployment

1. Upload the project files to the hosting account's web root (`htdocs` in this deployment).
2. Configure `admin_config.php` with the database host, database name, database username, database password, and admin session settings.
3. Configure `config.php` and the Google Drive integration according to the uploader's setup.
4. Create or verify the required database tables in phpMyAdmin.
5. Confirm the public gallery and upload workflow work before testing admin analytics.
6. Sign in to the admin area and open `/admin/analytics.php`.
7. Test the CSV export and check that timestamps are consistent with India Standard Time.

Do not publish real configuration files or credentials to GitHub. Keep private configuration files out of the repository.

## Security and Privacy Notes

- Never commit database passwords, API credentials, private keys, session secrets, or service-account files.
- Use prepared statements for SQL operations.
- Keep admin pages behind server-side authentication.
- Regenerate the session ID after login and destroy the session on logout.
- Apply appropriate session timeout and secure cookie settings.
- Restrict access to configuration files and backups.
- Keep PHP errors and stack traces out of public responses; log diagnostic details privately.
- This project records IP addresses, which can be personal data. Tell visitors what is collected, why it is collected, how long it is retained, and who can access it. Consider whether full IP retention is necessary and apply an appropriate retention policy.
- Use HTTPS for the deployed website.
- Back up the database and files before making changes.

## Troubleshooting

### Analytics page does not show new records
- Confirm `visitor-tracker.php` is included before any HTML output.
- Confirm the public page is receiving a `GET` request.
- Check the database credentials and table names.
- Review the hosting PHP error log without exposing credentials.

### Admin page returns HTTP 500
- Check for duplicate PHP function declarations and syntax errors.
- Review the hosting PHP error log.
- Restore the last known working backup if needed.

### Dates or times appear incorrect
- Check the PHP time zone (`Asia/Kolkata`).
- Check the MySQL session time zone.
- Confirm how timestamps are generated and stored before converting any existing records. Do not bulk-shift stored dates without confirming whether they are already in IST or UTC.

### CSV export fails
- Confirm the admin session is valid.
- Check the export endpoint and database query.
- Review server logs for the error details.

## Backup Checklist

Before changing code or database schema:

- Download a copy of the current project files.
- Export the MySQL database from phpMyAdmin.
- Keep private configuration files securely outside public GitHub repositories.
- Test changes on a backup or staging copy when possible.

## License

Choose and add a license before accepting reuse or contributions. If no license is included, others generally do not receive explicit permission to reuse, modify, or redistribute the code.

---

**Project:** Friends Group Durga Puja Photo Uploader  
**Hosting:** InfinityFree  
**Language:** PHP, HTML, CSS, JavaScript, SQL
