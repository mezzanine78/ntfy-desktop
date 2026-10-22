# Installing ntfy Desktop on Windows 11 (self-hosted server + access token)

This fork tracks the upstream project [simoneferrari/ntfy-desktop](https://github.com/simoneferrari/ntfy-desktop)
(MIT licence, see [LICENSE](LICENSE)). All credit for the app goes to its author. This guide only adds
step-by-step setup for a **self-hosted ntfy server that requires an access token**.

> This fork does not publish its own builds. Download installers from the
> **[upstream releases page](https://github.com/simoneferrari/ntfy-desktop/releases/latest)**.
> The app's built-in updater also pulls from upstream.

## 1. Requirements

| | |
|---|---|
| OS | Windows 11 (or Windows 10 1809+), x64 |
| Runtime | Nothing to install: the .NET 10 runtime is bundled in the release |
| Rights | Standard user. No administrator rights needed |
| Server | An ntfy server reachable over **https://** (the app refuses to send a token over plain http) |
| Token | An ntfy access token (`tk_...`) for a user with read access to your topics |

### Getting an access token (on the ntfy server)

If you run the server yourself, create a token for your user on the server host:

```sh
ntfy token add <username>            # prints tk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ntfy access <username> <topic> read   # make sure the user can read the topic
```

Or sign in to the server's web app, then open **Account → Access tokens → Create access token**.
Treat the token like a password: don't paste it into chats, tickets, scripts or Git repos.

## 2. Download and install

1. Open the [latest upstream release](https://github.com/simoneferrari/ntfy-desktop/releases/latest).
2. Pick one:
   - **Installer (recommended):** `NtfyDesktop-stable-Setup.exe`. It installs per user to
     `%LocalAppData%\NtfyDesktop`, adds a Start-menu shortcut and launches the app.
   - **Portable:** `NtfyDesktop-stable-Portable.zip`. Extract it anywhere and run `NtfyDesktop.exe`.
3. **SmartScreen:** the app isn't code-signed, so Windows may show *"Windows protected your PC"*.
   Click **More info → Run anyway**. To be careful, check the VirusTotal link in the release notes first,
   or check the file hash:

   ```powershell
   Get-FileHash "$env:USERPROFILE\Downloads\NtfyDesktop-stable-Setup.exe" -Algorithm SHA256
   ```

The app starts in the system tray (notification area). Double-click the tray icon to open the window.

## 3. Connect to your self-hosted server with a token

1. Open **Settings → Servers**.
2. Add a server (or edit the default `https://ntfy.sh` entry if you don't use it):
   - **URL:** `https://ntfy.example.com` (your server, **https** only, no topic in the URL).
   - **Authentication:** choose **Access token** and paste your `tk_...` token.
     (Username and password also works if your server uses it.)
   - Optionally give it a label.
3. Save. The token is encrypted with Windows DPAPI for your Windows user and stored in
   `%AppData%\NtfyDesktop\settings.json`. Only your Windows account on that PC can decrypt it.
4. In the left rail click **Add topic**, choose your server, enter the topic name
   (for example `alerts`), and optionally a display name. Repeat for each topic.
5. The topic's connection status should go green. Test it from any machine:

   ```powershell
   $h = @{ Authorization = "Bearer $env:NTFY_TOKEN"; Title = 'Test'; Priority = '4'; Tags = 'white_check_mark' }
   Invoke-RestMethod -Method Post -Uri 'https://ntfy.example.com/alerts' -Headers $h -Body 'Hello from PowerShell'
   ```

   A Windows toast should pop up and the message should show in the feed.

If you don't want messages to go through the public ntfy.sh server at all, delete the default
`https://ntfy.sh` server entry or just don't add topics to it. The app only connects to a server
once a topic uses it.

## 4. Notifications: priority, tags and clicks

- **Priority 1-5** sets the toast's sound and urgency (5 = urgent). Set a minimum priority per topic
  or globally in Settings.
- **Tags** (e.g. `warning`, `rotating_light`, `white_check_mark`) are shown as emoji.
- **Click URL**: clicking the toast opens the message's `click` link. Only http/https links are opened.
- **Action buttons**: `view` opens a link, `copy` copies text, and `http` asks for confirmation before sending.
- **Pause / Active hours**: pause toasts globally or per topic, or only show them in a time window.
  Priority 5 messages are still worth leaving on for alerts.
- Check **Windows Settings → System → Notifications → ntfy Desktop** is turned on, and that
  Do Not Disturb / Focus isn't hiding toasts.

## 5. Start with Windows

Turn on **Settings → Start with Windows**. This adds a per-user entry under
`HKCU\Software\Microsoft\Windows\CurrentVersion\Run`. Turn it off the same way.
Closing the window keeps the app running in the tray. To quit fully, right-click the tray icon → **Exit**.

## 6. Updates

The app checks GitHub Releases at startup and once a day, then shows **Restart & update** when a new
version is out. It downloads from **upstream** (`simoneferrari/ntfy-desktop`). You can switch the
automatic check off or check by hand in **Settings → Updates**. Re-running a newer `Setup.exe` also updates
in place and keeps your settings and tokens.

## 7. Uninstall

- **Installer version:** Windows **Settings → Apps → Installed apps → ntfy Desktop → Uninstall**.
- **Portable:** quit from the tray and delete the extracted folder.
- To also remove settings, message history and the encrypted token:

  ```powershell
  Remove-Item "$env:APPDATA\NtfyDesktop" -Recurse -Force
  ```

  Then revoke the token on the server (`ntfy token remove <username> <token>`, or in the web app
  under **Account → Access tokens**).

## 8. Troubleshooting

| Symptom | Fix |
|---|---|
| Topic stays red or "unauthorized" | Wrong or expired token, or the user lacks read access to the topic (`ntfy access <user>`). Re-paste the token in Settings → Servers. |
| Warning about http / token not sent | Use an `https://` URL. The app never sends credentials over plain http or ws. |
| Connects but no toasts | Check Windows notification settings and Focus, the topic's pause/active-hours, and the minimum-priority filter. |
| Behind a reverse proxy, never connects | The app uses WebSockets (`wss://<server>/<topic>/ws`). Make sure the proxy passes `Upgrade`/`Connection` headers. |
| Token in a different place | Use `--data-path <dir>` to keep settings in a custom folder (e.g. for a portable install). |

## Security notes (from a review of v0.75.0, Oct 2026)

- No telemetry. It only connects to the servers you add, GitHub Releases (update check), attachment URLs in
  messages, and `http` actions after you confirm.
- Tokens and passwords are DPAPI-encrypted at rest and only sent to their own server over TLS.
- The release isn't code-signed, and the bundled .NET runtime falls behind Microsoft's patches between
  upstream releases. Keep the app updated.
- Only open attachments from senders you trust. Saved attachments don't get the Windows
  "downloaded from the internet" mark.
