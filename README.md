# selfsign_cert
C, powershell, self-sign EXE for own testing code

# C 語言專案編譯與 PowerShell 數位簽署完整教學

本教學說明如何從頭建置一個具備自訂圖示的 C 語言 Windows 應用程式，並透過 PowerShell 建立帶有聯絡資訊（Email）與易記名稱（Friendly Name）的程式碼簽署憑證，最後利用批次檔自動進行數位簽署與驗證。

---

## 目錄
1. [環境需求與目錄結構](#1-環境需求與目錄結構)
2. [步驟一：準備資源與 C 語言原始碼編譯](#2-步驟一準備資源與-c-語言原始碼編譯)
3. [步驟二：使用 PowerShell 建立並匯入自訂憑證](#3-步驟二使用-powershell-建立並匯入自訂憑證)
4. [步驟三：撰寫自動化簽署批次檔 (sign.bat)](#4-步驟三撰寫自動化簽署批次檔-signbat)
5. [步驟四：整合自動化編譯與簽署腳本](#5-步驟四整合自動化編譯與簽署腳本)
6. [步驟五：數位簽署驗證與常見問題排解](#6-步驟五數位簽署驗證與常見問題排解)

---

## 1. 環境需求與目錄結構

### 軟體環境需求
- **GCC / MinGW**：用於編譯 C 語言程式碼及資源檔（`windres`）。
- **Windows SDK (SignTool)**：用於對可執行檔進行數位簽署（`signtool.exe`）。
- **PowerShell 5.1+**：用於產生自簽憑證。

### 專案目錄結構
```text
ProjectRoot/
│
├── ico/
│   └── app.ico                  # 應用程式圖示
├── selfsign_cert/
│   ├── make_cert.ps1             # 建立並匯出憑證的 PowerShell 腳本
│   └── sign.bat                  # 數位簽署作業批次檔
├── resource.rc                   # Windows 資源定義檔
├── main1.c          # 主程式碼 1
├── main1_with_ico.c # 主程式碼 2
└── build_and_sign.bat            # 一鍵編譯與簽署總腳本
```

---

## 2. 步驟一：準備資源與 C 語言原始碼編譯

### 1. 建立資源檔 (`resource.rc`)
定義應用程式圖示：
```rc
MAINICON ICON "ico/app.ico"
```

### 2. 使用 `windres` 編譯資源
利用 MinGW 提供的 `windres` 工具將 `.rc` 編譯為 `.o` 物件檔：
```cmd
windres resource.rc resource.o
```

### 3. 使用 GCC 編譯 EXE 可執行檔
使用 `-mwindows` 隱藏主控制台視窗，並以 `-finput-charset=UTF-8` 確保中文編碼無誤：
```cmd
gcc main1.c resource.o -o TOOGLE.exe -mwindows -finput-charset=UTF-8 -fexec-charset=UTF-8 -ladvapi32 -luser32
gcc main1_with_ico.c resource.o -o SET.exe -mwindows -finput-charset=UTF-8 -fexec-charset=UTF-8 -ladvapi32 -luser32
```

---

## 3. 步驟二：使用 PowerShell 建立並匯入自訂憑證

由於 Windows 限制程式碼簽署憑證（`CodeSigningCert`）建立時必須存放在「個人存放區（`cert:\CurrentUser\My`）」，因此腳本會先建立憑證，再自動將其複製到「受信任的根憑證授權單位（`Root`）」以確保系統信任該憑證。

### 自動化 PowerShell 腳本 (`selfsign_cert\make_cert.ps1`)

```powershell
# 1. 建立憑證（包含 Subject, Support Email 與 EKU 延伸屬性）
$cert = New-SelfSignedCertificate `
    -Subject "CN=NAME_OF_CERT, E=you@your_mail.com, O=Organisation, C=TW" `
    -TextExtension @(
        "2.5.29.17={text}email=you@your_mail.com",
        "2.5.29.37={text}1.3.6.1.5.5.7.3.3,1.3.6.1.5.5.7.3.4"
    ) `
    -CertStoreLocation "cert:\CurrentUser\My" `
    -KeyExportPolicy Exportable `
    -KeyUsage DigitalSignature, NonRepudiation `
    -Type CodeSigningCert `
    -NotAfter (Get-Date).AddDays(1095)

# 2. 設定憑證易記名稱 (Friendly Name)
$cert.FriendlyName = "Name of your cert"

# 3. 複製到受信任的根憑證授權單位存放區 (Root)
$rootStore = [System.Security.Cryptography.X509Certificates.X509Store]::new("Root", "CurrentUser")
$rootStore.Open([System.Security.Cryptography.X509Certificates.OpenFlags]::ReadWrite)
$rootStore.Add($cert)
$rootStore.Close()

# 4. 匯出成 .pfx 檔案儲存於桌面供 signtool 使用
$pwd = ConvertTo-SecureString -String "YourSecurePassword123" -Force -AsPlainText
Export-PfxCertificate -Cert $cert -FilePath "$HOME\Desktop\your_cert.pfx" -Password $pwd

Write-Host "憑證建立與匯出完成！" -ForegroundColor Green
```

---

## 4. 步驟三：撰寫自動化簽署批次檔 (`sign.bat`)

在簽署檔案時，必須指定 **`/fd sha256`**（檔案摘要演算法）與 **`/td sha256`**（時間戳記演算法），否則新版 SignTool 會拋出 `No file digest algorithm specified` 錯誤。

### 簽署腳本 (`selfsign_cert\sign.bat`)

```cmd
@echo off
chcp 65001 >nul
echo 正在對檔案進行數位簽署: %~1 ...

:: 檢查傳入參數
if "%~1"=="" (
    echo [錯誤] 未指定要簽署的 EXE 檔案！
    exit /b 1
)

:: 設定變數 (請確認 signtool.exe 與 .pfx 路徑)
set "SIGNTOOL=C:\Program Files (x86)\Windows Kits\10\bin\10.0.22621.0\x64\signtool.exe"
set "PFX_PATH=%USERPROFILE%\Desktop\your_cert.pfx"
set "PASSWORD=YourSecurePassword123"
set "TIMESTAMP_URL=http://timestamp.digicert.com"

:: 執行簽署指令
"%SIGNTOOL%" sign /f "%PFX_PATH%" /p "%PASSWORD%" /tr "%TIMESTAMP_URL%" /td sha256 /fd sha256 "%~1"

:: 檢查簽署結果
if %errorlevel% equ 0 (
    echo [成功] %~1 簽署完成。
) else (
    echo [失敗] %~1 簽署失敗，請檢查錯誤記錄。
    exit /b 1
)
```

---

## 5. 步驟四：整合自動化編譯與簽署腳本

### 重要觀念：使用 `call` 指令
在 Windows 批次檔（`.bat`）中，若要呼叫另一個 `.bat` 檔並在執行完後**回傳繼續執行後續指令**，必須在前面加上 **`call`**。否則控制權轉移後，主腳本會在子腳本結束時直接終止，導致第二個簽署動作無法被執行。

### 完整建置腳本 (`build_and_sign.bat`)

```cmd
@echo off
chcp 65001 >nul

:: 設定環境變數路徑
path=C:\MinGW\bin\;ico\;%PATH%

echo [1/3] 編譯資源檔...
windres resource.rc resource.o

echo [2/3] 編譯 C 語言程式碼...
gcc main1.c resource.o -o TOOGLE.exe -mwindows -finput-charset=UTF-8 -fexec-charset=UTF-8 -ladvapi32 -luser32
gcc main1_with_ico.c resource.o -o SET.exe -mwindows -finput-charset=UTF-8 -fexec-charset=UTF-8 -ladvapi32 -luser32

echo [3/3] 執行數位簽署...
:: 必須使用 call 呼叫子批次檔
call .\selfsign_cert\sign.bat SET.exe
call .\selfsign_cert\sign.bat TOOGLE.exe

echo.
echo ========================================
echo   所有程式建置與數位簽署已成功完成！
echo ========================================
echo.
pause
```

---

## 6. 步驟五：數位簽署驗證與常見問題排解

### 1. 驗證數位簽章指令
在 CMD 終端機執行以下指令：
```cmd
"C:\Program Files (x86)\Windows Kits\10\bin\10.0.22621.0\x64\signtool.exe" verify /pa /v SET.exe
```
* **預期結果**：終端機最後顯示 `Successfully verified: SET.exe`。

### 2. 常見問題與說明

| 問題現象 | 原因說明 | 解決方式 |
| :--- | :--- | :--- |
| **`No file digest algorithm specified`** | SignTool 缺乏 `/fd` 參數。 | 在 `signtool sign` 指令中加入 `/fd sha256`。 |
| **`新憑證只能安裝到我的存放區`** | `New-SelfSignedCertificate` 強制 `CodeSigningCert` 只能在 `My` 建立。 | 先建於 `cert:\CurrentUser\My`，再用 .NET API 複製到 `Root`。 |
| **副署中的電子郵件顯示「無法使用」** | 副署為第三方時間戳記伺服器（如 DigiCert）憑證，其本身無 Email 欄位。 | **此屬正常現象**，完全不影響簽署驗證與 SmartScreen 信任狀態。 |
| **第二個 `sign.bat` 未執行** | 批次檔呼叫子批次檔未加 `call`。 | 呼叫時加上 `call .\selfsign_cert\sign.bat ...`。 |
