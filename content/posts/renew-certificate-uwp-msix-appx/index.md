---
title: Renewing certificate for UWP package (MSIX, APPX)
date: 2020-12-24T14:52:47Z
summary: How I renewed expired certificate file of released and packaged UWP
  application that prevented users from installing it.
tags:
  - UWP
categories:
  - Programming
projects:
  - ipasim
cover:
  image: cover.png
  alt: Certificate + MSIX package
  relative: true
ShowToc: true
TocOpen: true
---

I have a UWP app package released and downloadable through GitHub releases.
(It's for my old [ipasim]({{< relref "/projects/ipasim" >}}) project if you are wondering.)
People download it quite often but I don't release new versions anymore (I have put this project on hold because it would be very time consuming to bring it to next level&mdash;right now it's more of a prototype).
Hence the certificate in the released package had expired because it's been over a year since I have built it.
But without valid certificate, the app cannot be installed.

**tl;dr**&mdash;here is a short PowerShell script that renews the `.cer` file and re-signs `.msix` package (you just need to fill in `-Subject` parameter and update path to `signtool.exe` if the provided one doesn't work for you):

```ps1
$cert = New-SelfSignedCertificate -Type Custom -Subject "CN=YOUR_NAME_HERE" -KeyUsage DigitalSignature -FriendlyName "devcert" -CertStore Cert:\CurrentUser\My\ -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.3", "2.5.29.19={text}")
Export-PfxCertificate -cert $cert -FilePath key.pfx -Password $(ConvertTo-SecureString -String "pwd" -Force -AsPlainText)
& 'C:\Program Files (x86)\Windows Kits\10\bin\10.0.18362.0\x64\signtool.exe' sign /fd sha256 /a /f key.pfx /p pwd .\*.msix
Remove-Item key.pfx
Export-Certificate -Cert $cert -FilePath (Get-ChildItem .\*.cer).name
Remove-Item $cert.PSPath
```

## Prerequisites

So, you've got yourself a UWP app package, that looks roughly like this:

```txt
Add-AppDevPackage.resources/
Dependencies/
Add-AppDevPackage.ps1
YourApp_1.0.0.0_Win32.cer
YourApp_1.0.0.0_Win32.msix
```

And when you try to install it through `Add-AppDevPackage.ps1` script, it says:

{{< samp >}}
Error: The developer certificate "YourApp_1.0.0.0_Win32.cer" has expired. One possible cause is the system clock isn't set to the correct date and time. If the system settings are correct, contact the app owner to re-create a package or bundle with a valid certificate.
{{< /samp >}}

Sure, you could build the app package again with new development certificate that can be [easily renewed in Visual Studio](https://docs.microsoft.com/en-us/previous-versions/br230260(v=vs.110)#renewing-a-certificate) (although this link is to outdated Microsoft Docs, the procedure should still work in Visual Studio 2019 as answered on [VS Developer Community Forums](https://developercommunity.visualstudio.com/content/problem/612872/create-test-certificate-option-missing-from-uwp-sd.html)).
However, you might not have source codes for the app or you simply don't want to build an old project because you might not even have all the necessary Visual Studio workloads installed.

No worries, the `.cer` file can be renewed and the `.msix` package re-signed without needing to re-build the app.
I have `.msix` package in my example but I believe the same steps would work for `.appx` package, as well.

1. Open elevated PowerShell console.

   ## Renewing certificate

2. **Determine certificate subject.**
   If you don't have source code, simply copy and rename `.msix` package to `.zip` and extract file named `AppxManifest.xml`.
   In that file, find `Publisher` attribute inside `<Identity>` tag.
   In my project, it looks like this:

   ```xml
   <Identity Name="0ee863f9-dcc5-4d3b-9c2a-457bcfafc07e" Publisher="CN=jjone" Version="1.0.1.0" ProcessorArchitecture="x86" />
   ```

   So, I will use `CN=jjone` as `-Subject` parameter in the next step.

3. **Create new development certificate** (see [Create a certificate for package signing](https://docs.microsoft.com/en-us/windows/msix/package/create-certificate-package-signing)):

   ```ps1
   $cert = New-SelfSignedCertificate -Type Custom -Subject "CN=jjone" -KeyUsage DigitalSignature -FriendlyName "devcert" -CertStore Cert:\CurrentUser\My\ -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.3", "2.5.29.19={text}")
   ```

   This will install the certificate into your computer's certificate store (specifically, `cert:\CurrentUser\My`).
   We will uninstall it when no longer needed in step 8.

   Also note that I store the certificate in PowerShell variable `$cert` for easier access in the following steps.

4. **Export `.pfx` file** that will be needed to sign `.msix` app package.

   ```ps1
   Export-PfxCertificate -cert $cert -FilePath key.pfx -Password $(ConvertTo-SecureString -String "pwd" -Force -AsPlainText)
   ```

   Note that the `.pfx` file has to be password-protected.
   Since this is just temporary development certificate and the `.pfx` file will be deleted later in step 8, I used not-so-secure string `pwd` as the password.

5. **Export `.cer` file** that will be needed by `Add-AppDevPackage.ps1` script when installing the app.

   Don't forget to use correct path to the certificate alongside your app package (here `YourApp_1.0.0.0_Win32.cer`).

   ```ps1
   Export-Certificate -Cert $cert -FilePath .\YourApp_1.0.0.0_Win32.cer
   ```

   This is just a public version of the certificate, so it is not password-protected.

   ## Re-signing app package

6. To sign the `.msix` file, you will need `signtool.exe` that comes with Windows 10 SDK (see [Sign an app package using SignTool](https://docs.microsoft.com/en-us/windows/msix/package/sign-app-package-using-signtool)).

   On my computer, I have installed Windows 10 SDK version `18362`, so this program is located at:

   ```txt
   C:\Program Files (x86)\Windows Kits\10\bin\10.0.18362.0\x64\signtool.exe
   ```

   You might need to use different path in the next step depending on Windows 10 SDK version *you* have installed.

7. **Re-sign `.msix` using `signtool.exe`** (ensure you use correct path to `signtool.exe` and name of your app package&mdash;here it is `YourApp_1.0.0.0_Win32.msix`):

   ```ps1
   & 'C:\Program Files (x86)\Windows Kits\10\bin\10.0.18362.0\x64\signtool.exe' sign /fd sha256 /a /f key.pfx /p pwd .\YourApp_1.0.0.0_Win32.msix
   ```

8. **Clean up** no-longer-needed certificate-related files.

   ```ps1
   Remove-Item key.pfx
   Remove-Item $cert.PSPath
   ```

   The latter uninstalls certificate installed in step 3.

9. That's it.
   Now you can try to install the app package again using `Add-AppDevPackage.ps1` script and it should succeed.
