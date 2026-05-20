SECURITY ADVISORY: Supply Chain Malware Detected — DO NOT USE

WARNING: This Repository Contains Malware

This repository contains a **supply chain malware dropper** embedded in the MSBuild `.vcxproj` file. **Building this project in Debug configuration will compromise your machine.** The entire repository appears to be **bait** — a seemingly functional DLL injector designed to lure developers into compiling and executing the hidden payload.

---

What Was Found

### Location
`DLL Injector V4/DLL Injector V4.vcxproj` — Lines 112 (Debug|x64 PreBuildEvent)

### Trigger
Building the project in `Debug|x64` configuration silently executes a multi-stage payload **on the developer's machine** during compilation.

### Malware Execution Chain (4 Stages)

```
┌─────────────────────────────────────────────────────────────────┐
│ STAGE 1: MSBuild PreBuildEvent (Batch Script)                   │
│ Triggered when user hits "Build Solution" (Ctrl+Shift+B)        │
│                                                                  │
│ • Creates: %TEMP%\z3IdPT\                                       │
│ • Writes: IkA3CyZgY.vbs via obfuscated echo concatenation       │
│ • Spawns: cscript //nologo "%TEMP%\z3IdPT\IkA3CyZgY.vbs"       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ STAGE 2: VBScript Decoder (IkA3CyZgY.vbs)                      │
│                                                                  │
│ • Uses MSXml2.DOMDocument.6.0 to decode Base64 → binary        │
│ • Uses ADODB.Recordset for binary chunk handling                │
│ • Writes decrypted payload to: %TEMP%\z3IdPT\PKCA.ps1           │
│ • Spawns: powershell.exe -ExecutionPolicy Bypass -File          │
│          "%TEMP%\z3IdPT\PKCA.ps1"                               │
│   with window hidden (SW_HIDE = 0)                              │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ STAGE 3: PowerShell Decryptor (PKCA.ps1)                        │
│                                                                  │
│ • Defines fn "wd9L7u6kgnx" (PBKDF2-SHA256 key derivation)       │
│ • Creates Rfc2898DeriveBytes(password, salt, iterations)         │
│ • Defines fn "ftFfhZMIpO1" (AES-CBC-256 decryption)             │
│ • Creates AesManaged in CBC mode with PKCS7 padding              │
│ • Creates decryptor from derived key + IV                       │
│ • Decrypts Base64 blob → raw bytes                              │
│ • Converts to string → [Array]::Reverse() to deobfuscate        │
│ • Creates alias "pWN" = Invoke-Expression                       │
│ • Calls pWN(decrypted_reversed_string)                          │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ STAGE 4: Unknown Final Payload                                   │
│                                                                  │
│ • Executed via Invoke-Expression on the deobfuscated string     │
│ • Base64 string "SW52b2tlLVY4cHJlc3Npb24=" → "Invoke-V8pression"│
│   (intentionally misspelled to evade string scanning)            │
│ • Likely: RAT, info-stealer, crypto-miner, or botnet agent      │
│ • Technique consistent with: Reflective .NET Assembly Load       │
└─────────────────────────────────────────────────────────────────┘
```

### Deobfuscated PowerShell Identifiers

| Obfuscated Variable | Deobfuscated Purpose |
|---------------------|---------------------|
| `$updghPRhQONPB` | AES decrypted data payload |
| `$yZhiBPRiGcccB` | AES IV |
| `$mbvHftYsefjJj` | `AesManaged.CreateDecryptor()` |
| `$mkKqjPpOucMnT` | Base64 decoded binary |
| `$funtounfdlccA` | Length offset calculation |
| `$mJijsJYeIWMcC` | `.ToCharArray()` result |
| `$TCBhHvHaeVTze` | `[System.Text.Encoding]::UTF8.GetString()` |
| `$fdedhvinigClb` | Decoded Base64 → `"Invoke-V8pression"` |
| `pWN` | Aliased to `Invoke-Expression` via `New-Alias` |

### Indicators of Compromise (IOCs)

| IOC | Value |
|-----|-------|
| Directory created | `%TEMP%\z3IdPT\` |
| VBS file dropped | `%TEMP%\z3IdPT\IkA3CyZgY.vbs` |
| PS1 file dropped | `%TEMP%\z3IdPT\PKCA.ps1` |
| Batch process spawned | `cscript.exe //nologo` |
| PowerShell process spawned | `powershell.exe -ExecutionPolicy Bypass -File` |
| COM object abused | `MSXml2.DOMDocument.6.0` |
| COM object abused | `ADODB.Recordset` |
| COM object abused | `Scripting.FileSystemObject` |
| COM object abused | `WScript.Shell` |
| .NET crypto abused | `System.Security.Cryptography.Rfc2898DeriveBytes` |
| .NET crypto abused | `System.Security.Cryptography.AesManaged` |

### Obfuscation Techniques Used

- **10 separate string variables** (`b`, `c`, `d`, `y`, `t`, `r`, `w`, `q`, `s`, `z`) each XOR'd/concatenated into the final payload
- **Arithmetic noise**: every string is wrapped in redundant bitwise operations like `((-534 -Bxor -534) -Band2*(534 -Band -534))` etc. that evaluate to 0 — padding to bloat the payload and evade signature scanning
- **Hidden PowerShell window**: `m.Run ..., 0, False` — window is hidden during execution
- **-ExecutionPolicy Bypass**: PowerShell runs with no restrictions
- **Base64 → AES-CBC-256**: dual-layer encryption requires both the key AND salt to decrypt
- **String reversal**: `[Array]::Reverse()` on the final command to break static analysis
- **Alias abuse**: `pWN` aliased to `Invoke-Expression` to hide the execution command
- **Misspelled evasion**: `"Invoke-V8pression"` instead of `"Invoke-Expression"` — evades simple string searches

---

 Why This Looks Like Bait

The codebase has several hallmarks of a honeypot targeting cheating/security tool developers:

1. **Hardcoded Chinese user paths**: `C:\Users\lisongqian\source\repos\Ldll\...` and `C:\Users\YOURNAME\...` — suggests original source repo
2. **Chinese UI text**: `设置Hook`, `卸载Hook`, `查看`, `进程`, `模块` — targets Chinese-speaking developers
3. **README promises advanced features** (ManualMapping, Thread Hijacking, QueueUserAPC, KernelCallback, etc.) that **don't exist in the code** — bait to make the project look sophisticated
4. **README describes PDB downloading for `ntdll.dll`** — a feature that would appeal to cheat developers wanting symbol resolution, but isn't implemented
5. **Only one injection method is implemented** (basic `CreateRemoteThread` + `LoadLibraryW`) — minimal functionality, maximum bait
6. **The target process is `spoolsv.exe`** — a SYSTEM-level Windows service, making the injector seem powerful
7. **GitHub topics/labels**: Likely tagged for "game hacking", "cheat", "injector" to attract the intended victim profile

---

If You Have Built This Project

If you compiled this project on your machine:

1. **Check if `%TEMP%\z3IdPT\` exists** on your system
2. **Check for `IkA3CyZgY.vbs`** or **`PKCA.ps1`** in that directory
3. **Check PowerShell execution logs**: `Get-WinEvent -LogName "Windows PowerShell" | Where-Object { $_.Message -like "*IkA3CyZgY*" }`
4. **Check Event Viewer**: Windows Logs → Application for `cscript.exe` or suspicious PowerShell execution
5. **Run antivirus scan**: Windows Defender Offline Scan or your preferred AV
6. **Monitor for**: Unusual outbound network connections, new startup entries, unexpected processes

---

 Repository Details
- **Commit analyzed**: `b4a735eb22ad3a6d33a2840f1eca042d28e8b552`
- **Affected file**: `DLL Injector V4/DLL Injector V4.vcxproj`
- **Affected configuration**: `Debug|x64` only (lines 112)
- **Other configurations**: `Release|Win32`, `Release|x64`, `Debug|Win32` — clean but misconfigured (`Console` subsystem instead of `Windows`)

---
