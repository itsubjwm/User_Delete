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
$allProfiles = Get-CimInstance Win32_UserProfile | Where-Object { -not $_.Special -and$_.LocalPath -notlike "C:\Windows*" }
$userProfiles = [System.Collections.Generic.List[PSCustomObject]]::new()

foreach ($prof in $allProfiles) {
    try {
        $sid = New-Object System.Security.Principal.SecurityIdentifier($prof.SID)
        $ntAccount =$sid.Translate([System.Security.Principal.NTAccount]).Value
    }
    catch {
        $ntAccount = "Account Unknown (SID: $($prof.SID))"
    }

    # Masukkan semua profil ke dalam daftar pilihan
    $userProfiles.Add([PSCustomObject]@{
        "Nama Akun / Domain" = $ntAccount
        "Path Folder"        = $prof.LocalPath
        "SID"                = $prof.SID
    })
}

if ($userProfiles.Count -eq 0) {
    Write-Host "Tidak ditemukan profile user di komputer ini." -ForegroundColor Yellow
    exit
}

# Membuka jendela GUI interaktif untuk memilih multiple user dari seluruh daftar
Write-Host "Membuka jendela pilihan profile... Silakan pilih user yang ingin dihapus." -ForegroundColor Cyan
$selectedProfiles =$userProfiles | Out-GridView -Title "Pilih User yang ingin dihapus (Tekan Ctrl/Shift untuk pilih banyak, lalu klik OK)" -OutputMode Multiple

# Jika user membatalkan atau tidak memilih
if ($null -eq $selectedProfiles -or$selectedProfiles.Count -eq 0) {
    Write-Host "Tidak ada profile yang dipilih. Proses dibatalkan." -ForegroundColor Yellow
    exit
}

Write-Host "`nMemulai proses penghapusan profile yang dipilih..." -ForegroundColor Cyan
$index = 1

foreach ($sel in $selectedProfiles) {
    $accountName = $sel."Nama Akun / Domain"
    $profilePath = $sel."Path Folder"
    $folderName = Split-Path $profilePath -Leaf

    # Pengaman: Jangan hapus folder sistem penting meskipun terpilih
    if ($folderName -in @("Public", "Default", "Default User", "All Users")) {
        Write-Warning "[$index/$($selectedProfiles.Count)] Melewati folder sistem penting: '$folderName'."
        $index++
        continue
    }

    try {
        $profile = Get-CimInstance Win32_UserProfile -ErrorAction SilentlyContinue |
            Where-Object { $_.LocalPath -ieq $profilePath }
        
        if ($null -ne $profile) {
            if (-not $profile.Loaded) {
                $profile | Remove-CimInstance
                Write-Host "[$index/$($selectedProfiles.Count)] Profile '$accountName' ($folderName) berhasil dihapus via WMI." -ForegroundColor Green
            }
            else {
                Write-Warning "[$index/$($selectedProfiles.Count)] Profile '$accountName' ($folderName) sedang digunakan. Mencoba menghapus folder fisik..."
                if (Test-Path $profilePath) {
                    Remove-Item -Path $profilePath -Recurse -Force -ErrorAction SilentlyContinue
                    Write-Host "Folder profile '$accountName' ($folderName) berhasil dihapus paksa." -ForegroundColor Yellow
                }
            }
        }
        else {
            if (Test-Path $profilePath) {
                Remove-Item -Path $profilePath -Recurse -Force
                Write-Host "[$index/$($selectedProfiles.Count)] Folder profile '$accountName' ($folderName) dihapus manual." -ForegroundColor Yellow
            }
            else {
                Write-Warning "[$index/$($selectedProfiles.Count)] Profile atau folder '$accountName' tidak ditemukan."
            }
        }
    }
    catch {
        Write-Error "Gagal menghapus profile '$accountName': $_"
    }
    $index++
}

Write-Host "`nSemua proses penghapusan profile yang dipilih selesai!" -ForegroundColor Green
```
