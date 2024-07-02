---
layout: post
title: Active Directory (AD) | Local Privilege Escalation (Part 2)
date: 2024-07-01 00:41:00 +0800
categories: [Cert, CRTP]
tags: [Cert, Active Directory]     # TAG names should always be lowercase
render_with_liquid: false
description: Designed to provide a comprehensive guide to Active Directory (AD) attack techniques
---

# Pre-quisite

- Mobexler
- Genymotion
- Frida (for Dynamic Analysis)

# Step-by-step

1. Setting Mobexler network
2. MobSF - uploading APK
3. Genymotion configuration :
- Download wifi adb from APKpure (https://apkpure.com/search?q=wifi+adb)
- Install the apk
- forget current wifi
- launch the wifi adb app
- Go to terminal and use command "adb connect x.x.x.x:xxxx

# Configure Frida

- Download frida server-android-arm64
- unxz file.xz
- sudo pip3 install frida-tools --upgrade
- sudo pip3 install frida --upgrade
- push frida server /data/local/
- frida-ps -Ua (list all app)
- (anti-root device)
frida -U -f com.android.insecurebankv2 --codeshare dzonerzy/fridantiroot --no-pause

# Configure Buprsuite

- export burp.der
- drag to mobexler
- change .der to .perm

```jsx
openssl x509 -inform DER -in burp.der -out burp.pem
```

- read hash pem file

```jsx
openssl x509 -inform PEM -subject_hash_old -in burp.pem | head -1
```

- change burp.perm to the hash name

```jsx
cp burp.pem 9a5ba575.0
```

- push the file to genymotion

```jsx
adb push 9a5ba575.0 /sdcard/Download
adb root;adb remount
```

- check the file

<aside>
💡 adb shell /sdcard/Download

</aside>

- move the file to cacerts
cp 9a5ba575.0 /system/etc/security/cacerts
- chmod
chmod 644 /etc/security/cacerts/9a5ba575.0
- add new listener to burp. add ip windows and port 8080

phone

- check cert at setting/security
- go to wifi setting, manually change the proxy and port (same with burp listener)