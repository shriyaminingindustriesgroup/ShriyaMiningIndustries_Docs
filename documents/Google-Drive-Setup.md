# Google Drive: how the app is linked to it, and how it was set up

Set up 20 Nov 2026 with the owner's personal Google account. Everything here is **free**: there's no billing account
in Google and nothing new in Azure. The architecture is in [ARCHITECTURE.md §11](../ARCHITECTURE.md); this is the
step-by-step record.

**Audience:**
- **Section 1:** anyone who asks "how did you link Google Drive to your web app?".
- **Sections 2 to 7:** whoever sets it up again (a new Google account, the move to a company account) or fixes it.

This repo is public, so **no secret values appear here**. The document names where each value is kept, never the
value itself.

---

## 1. How it works, in plain words

The app (`https://app.shriyamining.com`) uses Google Drive for two things:

| What | How the app reaches Drive | What Google lets it do |
|---|---|---|
| **Saving files:** invoice PDFs on every status change, uploaded PO files | An Admin clicks **Connect** once and signs in with Google (OAuth 2.0). The app keeps a renewable access pass, called a refresh token, and stores it encrypted in its database. | Permission **`drive.file`** only: the app sees and changes **only the files and folders it created itself**, nothing else in the Drive. |
| **Reading photos** to attach to invoice emails | A Google **service account** (a robot account, `photo-reader@…`) that the owner shared the photo folders with as **Viewer**. | Read-only, and only the folders shared with it. |

- **Where files go** (the app creates these folders itself):
  - invoices: `Invoice / <Buyer> / <financial year> / Invoice_22_InReview_MMDDYYYYHHMMSS.pdf`. Only the latest
    status's file is kept; the previous one goes to the bin.
  - PO files: `PO's / <Buyer> / <year> / <PO>_<Buyer>_<time>.<ext>`, in the format they were uploaded.
- **Nobody signs in to Google when sending or saving.** The one-time connection keeps working: the app renews its
  own access. Only an Admin connects, and only once.
- **Safety:**
  - The narrow `drive.file` permission means Google doesn't need to review the app, and the app can't see the
    owner's other files.
  - The client secret, the photo-reader key and the encryption key live only in Azure's settings, never in code or
    in this repo.
  - The stored Google pass is encrypted (AES-GCM) with a different key in test and in production.
- **Test and production are separate:** each has its own connection, its own folders and its own encryption key.
- **Cost:** $0. The Google Drive API is free at this volume, files use the Google account's free 15 GB, and Azure
  adds no cost.

```
Admin ──"Connect"──► Google sign-in (drive.file) ──► back to app page /cloud-connect/google
                                                     └─► API swaps the code for a refresh token, stores it encrypted
Invoice status changes ──► browser makes the PDF ──► API ──(refresh token)──► Google Drive: Invoice/Buyer/Year/…
Sender: "Add photos ▾" ──► API ──(service account, read-only)──► shared photo folder ──► picked photos attached
```

---

## 2. Google Cloud project

1. Open **https://console.cloud.google.com**, signed in with the Google account that will own the Drive folders.
   Accept the terms if asked.
2. In the top bar, open the **project picker** (left of the search box) → **New project**.
   - Project name: `ShriyaMining-Cloud`.
   - Location: *No organization*.
   - **Create**.
   - The project ID became `shriyamining-cloud`. Select the project so the top bar shows it.
3. ☰ menu → **APIs & Services** → **Library** → search `Google Drive API` → **Enable**.
4. **Don't add a billing account.** Without one, Google can't charge anything.

## 3. Sign-in client (saving files)

Direct link to this area: **https://console.cloud.google.com/auth/clients?project=shriyamining-cloud**. In the menu
it's ☰ → **APIs & Services** → **OAuth consent screen**, which newer screens call **Google Auth Platform**.

### 3.1 Consent screen

**Get started**, then:
- **App information:** App name `Shriya Mining Invoices`, User support email = the owner's email → **Next**.
- **Audience:** **External** → **Next**.
- **Contact information:** the owner's email → **Next**.
- Tick the policy → **Continue** → **Create**.

### 3.2 The one permission

1. Left menu **Data access** → **Add or remove scopes**.
2. In the filter box, type `drive.file`.
3. Tick **`.../auth/drive.file`** ("See, edit, create and delete only the specific Google Drive files you use with
   this app"). Not `drive` and not `drive.readonly`.
4. **Update** → **Save**. It must appear under *non-sensitive scopes*, with nothing under *sensitive scopes*.

### 3.3 Branding

*Publish app* stays greyed out until this page is saved.

Left menu **Branding**:

| Field | Value |
|---|---|
| App name | `Shriya Mining Invoices` |
| User support email | the owner's email |
| App logo | **leave empty**: a logo makes Google review the app |
| Application home page | `https://app.shriyamining.com` |
| Application privacy policy link | `https://app.shriyamining.com/privacy` |
| Application terms of service link | `https://app.shriyamining.com/terms` |
| Authorized domains | `shriyamining.com` |
| Developer contact email | the owner's email |

Then **Save**.

### 3.4 Publish

1. Left menu **Audience** → **Publish app** → **Confirm**.
2. *Publishing status* must say **In production**.
   - **Why:** in *Testing*, Google disconnects the app every 7 days.
   - With only `drive.file` and no logo, no verification is needed. Ignore "submit for verification".
3. Don't click *Back to testing*.

### 3.5 Create the client

1. Left menu **Clients** → **+ Create client**.
2. Application type: **Web application**. Name: `Shriya Mining app`.
3. **Authorized JavaScript origins:** leave empty.
4. **Authorized redirect URIs:** **+ Add URI** for each of these four:

   ```
   https://app.shriyamining.com/cloud-connect/google
   https://www.shriyamining.com/cloud-connect/google
   https://gray-meadow-0ebc61d10.6.azurestaticapps.net/cloud-connect/google
   https://gray-meadow-0ebc61d10-test.centralus.6.azurestaticapps.net/cloud-connect/google
   ```

   These are app **pages**, not `/api` addresses. See section 7: an `/api` return address fails on Static Web Apps.
   A new site address (for example a custom test domain) must be added here, or Google answers
   `redirect_uri_mismatch`.
5. **Create** → **Download JSON**. The file is `client_secret_….json` and holds the Client ID and Client secret.
   Newer screens show the secret only once.

## 4. Service account (reading the photo folders)

1. ☰ → **IAM & Admin** → **Service accounts** → **+ Create service account**:
   - Name `photo-reader` (the ID fills in by itself). Description: *Reads the photo folder for the email pop-up*.
   - **Create and continue** → *Grant access*: leave empty → **Continue** → **Done**.
2. Copy its email: `photo-reader@shriyamining-cloud.iam.gserviceaccount.com`.
3. Click it → **Keys** → **Add key** → **Create new key** → **JSON** → **Create**. A file named
   `shriyamining-cloud-<id>.json` downloads.
4. Share the photo folders with it in **https://drive.google.com**:
   1. Right-click the folder → **Share**.
   2. Paste the `photo-reader@…` email and set the role to **Viewer**.
   3. Untick **Notify people** → **Share** (and **Share anyway** if Google warns it's outside the organisation).

   | Environment | Folder |
   |---|---|
   | Production | `InvoiceProductImages` |
   | Test | `InvoiceProductImagesTest` |

   To check: open **Share** again; `photo-reader@…` must be listed as **Viewer**.

## 5. Azure settings (PowerShell + Azure Portal)

The app reads four settings from the Static Web App **ShriyaMiningIndustries-ui** (resource group
`ShriyaMiningIndustries`), one set for **Production** and one for **test**:

| Setting | Value | Test vs production |
|---|---|---|
| `CLOUD_GOOGLE_CLIENT_ID` | `client_id` from the client JSON (ends `.apps.googleusercontent.com`) | same |
| `CLOUD_GOOGLE_CLIENT_SECRET` | `client_secret` from the client JSON (starts `GOCSPX-`) | same |
| `PHOTO_GOOGLE_SERVICE_ACCOUNT_KEY` | the service-account key file as **one Base64 line** (about 3,000 characters starting `ewog`) | same |
| `CLOUD_TOKEN_ENCRYPTION_KEY` | a random 32-byte key, Base64 (about 44 characters, usually ending `=`) | **different in each** |

### 5.1 Move the downloaded files somewhere private first

The two JSON files are secrets. Keep them out of OneDrive and out of any git folder: OneDrive starts uploading a
file the moment it lands in a synced folder. In PowerShell:

```powershell
New-Item -ItemType Directory -Force C:\SecretsTemp | Out-Null
Move-Item "$env:USERPROFILE\Downloads\client_secret_*.json" C:\SecretsTemp\
Move-Item "$env:USERPROFILE\Downloads\shriyamining-cloud-*.json" C:\SecretsTemp\
Get-ChildItem C:\SecretsTemp          # should list the two files
```

Change the source paths if the files were saved elsewhere. `C:\SecretsTemp` isn't synced.

### 5.2 Produce each value (PowerShell)

`| clip` copies a value to the clipboard without showing it. Without `| clip`, it prints the value so you can copy it
by hand. Run each line just before pasting, because the clipboard holds only the last thing copied.

```powershell
# CLOUD_GOOGLE_CLIENT_ID
(Get-Content (Get-ChildItem C:\SecretsTemp\client_secret_*.json).FullName | ConvertFrom-Json).web.client_id | clip

# CLOUD_GOOGLE_CLIENT_SECRET
(Get-Content (Get-ChildItem C:\SecretsTemp\client_secret_*.json).FullName | ConvertFrom-Json).web.client_secret | clip

# PHOTO_GOOGLE_SERVICE_ACCOUNT_KEY (the key file as one Base64 line)
[Convert]::ToBase64String([IO.File]::ReadAllBytes((Get-ChildItem C:\SecretsTemp\shriyamining-cloud-*.json).FullName)) | clip

# CLOUD_TOKEN_ENCRYPTION_KEY: run once for Production and AGAIN for test (each run makes a new key)
$b = New-Object byte[] 32; [Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($b); $k = [Convert]::ToBase64String($b); $k | clip
```

- **If a paste comes out empty:**
  - Check the clipboard with `Get-Clipboard`, or paste into Notepad first.
  - For the long photo key, this opens it in Notepad (Ctrl+A, Ctrl+C):

    ```powershell
    [Convert]::ToBase64String([IO.File]::ReadAllBytes((Get-ChildItem C:\SecretsTemp\shriyamining-cloud-*.json).FullName)) | Out-File C:\SecretsTemp\photo-key.txt -Encoding ascii -NoNewline; notepad C:\SecretsTemp\photo-key.txt
    ```

- **Never screenshot a window that shows a value.** If one is exposed, replace it (section 6.3).

### 5.3 Enter them in the Azure Portal

1. **https://portal.azure.com** → search **Static Web Apps** → **ShriyaMiningIndustries-ui** → **Settings** →
   **Environment variables**.
2. **Environment** dropdown → **Production**.
3. For each setting: **+ Add** → Name → click in Value → **Ctrl+V** → **Apply** (in the box).
4. Then click **Apply at the bottom of the page** → **Confirm**.
   - The box's Apply only adds the setting to the list on screen; **the page's Apply saves it**. Skipping it lost
     two settings the first time.
5. Press **F5** and check all four names are still listed.
6. Switch the dropdown to **test** and repeat: the same values, except a **new** `CLOUD_TOKEN_ENCRYPTION_KEY`.

### 5.4 Or all at once with the Azure CLI (PowerShell)

This does sections 5.2 and 5.3 in one go, prints only the setting **names**, and makes a different encryption key
for each environment. It needs the Azure CLI and `az login` first.

```powershell
az login
$ErrorActionPreference = 'Stop'
$app = 'ShriyaMiningIndustries-ui'; $rg = 'ShriyaMiningIndustries'
$client = Get-Content (Get-ChildItem C:\SecretsTemp\client_secret_*.json).FullName | ConvertFrom-Json
$photoKey = [Convert]::ToBase64String([IO.File]::ReadAllBytes((Get-ChildItem C:\SecretsTemp\shriyamining-cloud-*.json).FullName))
function New-Key { $b = New-Object byte[] 32; [Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($b); [Convert]::ToBase64String($b) }
$common = @("CLOUD_GOOGLE_CLIENT_ID=$($client.web.client_id)", "CLOUD_GOOGLE_CLIENT_SECRET=$($client.web.client_secret)", "PHOTO_GOOGLE_SERVICE_ACCOUNT_KEY=$photoKey")
az staticwebapp appsettings set -n $app -g $rg --setting-names @($common + "CLOUD_TOKEN_ENCRYPTION_KEY=$(New-Key)") --output none
az staticwebapp appsettings set -n $app -g $rg --environment-name test --setting-names @($common + "CLOUD_TOKEN_ENCRYPTION_KEY=$(New-Key)") --output none
Remove-Variable client, photoKey, common
```

**Check the names only, never the values.** These are safe to run any time:

```powershell
az staticwebapp appsettings list -n ShriyaMiningIndustries-ui -g ShriyaMiningIndustries --query "keys(properties)" -o tsv | Select-String 'CLOUD_|PHOTO_'
az staticwebapp appsettings list -n ShriyaMiningIndustries-ui -g ShriyaMiningIndustries --environment-name test --query "keys(properties)" -o tsv | Select-String 'CLOUD_|PHOTO_'
```

> **Note:** `az staticwebapp appsettings set` replaces **all** of an environment's settings with the list given, on
> some CLI versions. Before using 5.4 on an environment that already has other settings (email, database), check
> with `az staticwebapp appsettings set --help`, or use the Portal (5.3), which only adds.

### 5.5 Clean up

```powershell
Remove-Item C:\SecretsTemp -Recurse -Force; cls
```

- Close Notepad **without saving**.
- If a JSON file ever sat in a OneDrive folder, delete it from **https://onedrive.live.com** → **Recycle bin**
  as well.
- Empty the Windows Recycle Bin.
- Copy any ordinary word to clear the clipboard.

## 6. In the app (once per environment, test first)

### 6.1 Connect and create the folders

1. Sign in as an Admin.
   - Test: `https://gray-meadow-0ebc61d10-test.centralus.6.azurestaticapps.net`
   - Production: `https://app.shriyamining.com`
2. **Admin → Company profile → Cloud storage**.
3. **Google Drive → Connect.** Google's page opens:
   - pick the account that owns the folders;
   - Google shows *"…see, edit, create and delete only the specific Google Drive files you use with this app"* →
     **Continue / Allow**;
   - you come back to Cloud storage, where Google Drive shows **Connected** and the account.
4. **Create Invoice folder** and **Create PO's folder.**
   - The app must create these itself: `drive.file` can't write into folders made by hand.
   - Test and production share one Google account, so rename test's folders in Drive, e.g. `Invoice-Test`. The app
     tracks folders by ID, so renaming and moving them is safe.
5. Set **Year starts on** (default 1 April) and **Year folder** label (`2026` or `2026-27`).
6. Tick **Save invoices on every status change** → **Save archive settings**.

### 6.2 Photo folders

1. **Photo folders** → paste the folder's link (open it in Drive and copy the address bar, or *Share → Copy link*).
2. **Check**: shows how many images were found and the newest date.
3. **Check and save.**

Use `InvoiceProductImagesTest` on test and `InvoiceProductImages` on production.

### 6.3 Checking it works

- **Invoices:** change an invoice's status. Within a minute the PDF appears in `Invoice / <Buyer> / <Year>`, and the
  earlier status's file is in the Drive bin. Failed or waiting saves are listed on Cloud storage, with **Retry**.
- **Photos:** in *Send invoice for approval* → **Add photos ▾** → the Drive folder → pick photos.
- **Upload a PO:** the file appears in `PO's / <Buyer> / <Year>`.

### 6.4 Replacing the client secret

Do this if the secret was exposed, for example shown in a screenshot or chat.

1. **https://console.cloud.google.com/auth/clients?project=shriyamining-cloud** → **Shriya Mining app** →
   **+ Add secret**.
2. Put the new secret in `CLOUD_GOOGLE_CLIENT_SECRET` in **Production and test**. Remember the page-bottom
   **Apply**.
3. Back in Google, **Disable** then **Delete** the old secret.

The Drive connection keeps working; there's no need to reconnect.

---

## 7. Problems we actually hit, and the fixes

| Problem | Cause | Fix |
|---|---|---|
| **Publish app** greyed out | Branding page not saved | Fill in Branding (3.3), Save, then Publish |
| Settings vanished after adding them | Only the box's **Apply** was clicked | Click **Apply at the bottom of the page** → Confirm, then F5 to check |
| Clipboard paste empty in Azure | Something else was copied in between, or `Set-Clipboard` didn't take | Use `\| clip`, check with Notepad or `Get-Clipboard`, or print the value and copy by hand |
| Client secret visible in a screenshot | Screenshot of PowerShell output | Replace the secret (6.4). Share only screens with names, never values |
| Secret files being uploaded by OneDrive | They downloaded into a synced folder | Move them to `C:\SecretsTemp` (5.1); delete them from OneDrive's recycle bin |
| **HTTP 500** when Google returned to the app | Azure Static Web Apps answers 500 to any **`/api`** request with a `code=` parameter, and Google always returns one | Google returns to the app **page** `/cloud-connect/google`, which passes the code on to the API. The old `/api/cloud-connections/google/callback` addresses were removed from the client |
| **`redirect_uri_mismatch`** | The return address isn't in the client's list, or isn't saved yet | Add the exact address under **Authorized redirect URIs** (not JavaScript origins): `https://`, no trailing `/`. **Save**, wait about 5 minutes, start again from the app. Google's *error details* link shows the address the app sent |
| App can't write into the hand-made `PO's` folder | `drive.file` only allows folders the app created | Use **Create PO's folder** in the app |
| Card shows **Needs reconnecting** | Google withdrew access (password change, app removed under the Google account's *Third-party connections*) | An Admin clicks **Connect** again; failed saves are retried |

## 8. Moving to a company Google account later

1. Keep the project and client. Add the company account as **Owner** of the Cloud project: IAM & Admin → IAM →
   **Grant access** → role *Owner*.
2. In the app: **Disconnect**, then **Connect** with the company account. Create new Invoice and PO's folders.
3. Share the new photo folders with `photo-reader@…`. A Google Workspace admin must allow sharing with outside
   accounts, or use a Shared Drive.
4. Files already saved stay in the personal account. Google can't transfer ownership from a personal account to a
   company one, so copy them if they're needed.
5. **Don't change `CLOUD_TOKEN_ENCRYPTION_KEY`.** If it changes, the stored connection can't be read and the app
   must be connected again.

## 9. Changelog

| Date | Change |
|---|---|
| 20 Nov 2026 | Google project, `drive.file` client (In production), `photo-reader` service account, four Azure settings per environment; connected on test and production. Return address moved from `/api/cloud-connections/google/callback` to the page `/cloud-connect/google` (SWA 500 on `/api?code=`). |
| 20–21 Nov 2026 | Photo folders (`InvoiceProductImages`, `InvoiceProductImagesTest`) shared with `photo-reader` and added in the app. |
| 21 Nov 2026 | Archive by status into `Invoice / Buyer / Year`; PO files into the app's own `PO's / Buyer / Year` (Create PO's folder). |
