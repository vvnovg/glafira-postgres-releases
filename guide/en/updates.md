# Updates and diagnostics

This chapter covers how to update the app, the **About** screen, the diagnostics archive and how to support the project.

<a id="updates"></a>

## Updates

When it starts, the app checks once whether a new version is available. If there is one, a notice above the tabs gives its version. **Details** opens the **About** screen. If the check fails, for example without internet access, no message is shown.

To check manually:

1. Open **About** at the bottom of the left panel.
2. In **Updates**, click **Check for updates**.
3. If a new version is available, the screen shows its number and release notes. Click **Install and restart**.

The app downloads the update, checks its signature, installs it and restarts. Open connections are closed before the restart; connection profiles, settings and other data are kept. A download interrupted by a network problem is retried automatically, up to three attempts in total. If all three attempts fail, the app reports **The update download was interrupted — check your network and try again**. If the signature check fails, the app reports **The update signature failed verification — the update was not installed**.

Installing may ask for an administrator password, because the app is installed for all users of the computer: by the `.pkg` installer on macOS, the `.msi` installer on Windows, or a `.deb` or `.rpm` package on Linux. Users without administrator rights cannot update such an installation.

Updates are downloaded from the releases page on GitHub. If the check fails with **Failed to check for updates**, check your network or proxy. **Updates are not configured in this build** means that this build was made without the updater key, for example a development or test build. In that case, download new versions from the releases page: https://github.com/vvnovg/glafira-postgres-releases/releases

An organization can turn off update checks through a policy, or point them to an internal mirror. When checks are turned off, the **Updates** section says **Updates are managed by your organization** after you click **Check for updates**. Like **Updates are not configured in this build**, this message appears only after the click; the check at startup shows nothing in these cases.

<a id="about"></a>

## About

Open **About** at the bottom of the left panel. On macOS, it is also in the Help menu as **About Glafira**.

The screen shows a short description of the app and:

- **Version** — the version number, and **Build** — the identifier of the source code it was built from. Include both in a support request.
- **System** — the operating system and processor architecture.
- **License** — Apache 2.0; the link opens the license text in the browser.
- **Organization policy** — **set** or **none**. Errors in the policy are listed below.
- **Data:** — the folder with connection profiles and settings.
- **Logs:** — the folder with log files.

The **Updates** and **Diagnostics** sections are also on this screen.

<a id="diagnostics"></a>

## Diagnostics archive

The diagnostics archive collects what support needs to understand a problem. To create it:

1. Open **About**.
2. In **Diagnostics**, click **Collect diagnostics**. On macOS, you can also choose **Collect Diagnostics…** in the Help menu.
3. Choose where to save the file. The suggested name is `diagnostics-<date>.zip`.

When the archive is saved, the screen shows its path. The ZIP file contains:

- `info.json` — the app version and build, the operating system, the data and log folder paths and the policy status;
- `connections.json` — connection profiles: names, environments, connection kinds, node and Patroni REST API addresses, user names, databases, certificate file paths, member address overrides and the SSH tunnel settings of each profile (bastion, user, port, key file path);
- `logs/` — the app log files and the SSH tunnel logs: `ssh-<connection id>.log` and `ssh-<connection id>.1.log` (the previous run), including the `ssh-unsaved-….log` files left by connection checks.

The archive contains no passwords or key passphrases, neither for PostgreSQL nor for SSH: they are kept in the system secret store, and passwords inside Patroni API addresses are removed. Certificate files and private keys are not included either, only the path to the key file. Host names, user names and file paths are stored as plain text, and the logs contain node addresses from PostgreSQL and Patroni error messages and the output of `ssh`. Look through the archive before you send it outside your company.

The app sends nothing by itself. Attach the archive to your support request yourself.

<a id="support"></a>

## Support the project

Glafira is free and open source under the Apache License 2.0. Donations keep it going. **Support the project** at the bottom of the left panel, on the **About** screen and, on macOS, in the Help menu opens the support page in the browser: https://github.com/vvnovg/glafira-postgres-releases/blob/main/SUPPORT.md
