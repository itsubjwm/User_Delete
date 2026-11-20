# Delete Multiple User

Script PowerShell untuk memindai dan menghapus beberapa profil pengguna (*user profile*) lokal di Windows secara massal.

## Script PowerShell

Klik ikon **Copy** di pojok kanan atas blok kode di bawah ini untuk menyalin seluruh kode:

```powershell
# Pastikan dijalankan sebagai Administrator
if (-not ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] "Administrator")) {
    Write-Error "Script ini harus dijalankan sebagai Administrator."
    exit
}

Write-Host "Memindai seluruh profil user di komputer..." -ForegroundColor Cyan

# Ambil semua profil user di komputer (kecuali profil khusus sistem Windows)
$allProfiles = Get-CimInstance Win32_UserProfile | Where-Object { -not $_.Special -and$_.LocalPath -notlike "C:\Windows\*" }
```
