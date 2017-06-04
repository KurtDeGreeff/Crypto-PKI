# PowerShell Protect-CmsMessage Example Code, Limitations and Errors

The PowerShell **Protect-CmsMessage** and **Unprotect-CmsMessage** cmdlets provide an easy way to use RFC 5652 public key encryption of your data. This article gives example PowerShell code to use Protect-CmsMessage, as well as a discussion about limitations, issues, problems, error messages, and so on.

## Background
[RFC 5652](https://www.google.com/search?as_q=&as_epq=rfc+5652) describes how to encrypt and sign messages using public/private key pairs in a vendor-neutral way. These messages conform to the Cryptographic Message Syntax (CMS) as defined by the RFC. The Protect-CmsMessage and Unprotect-CmsMessage cmdlets are a PowerShell implementation of CMS. Data encrypted in PowerShell this way can be decrypted, for example, by [OpenSSL on Linux](http://blogs.msdn.com/b/powershell/archive/2015/06/09/powershell-the-blue-team.aspx).

These cmdlets make it very easy to strongly encrypt passwords, credit card numbers, firewall configuration scripts, and other secrets which can then be stored securely or sent over the Internet with Invoke-WebRequest or Send-MailMessage. When combined with PowerShell scripting for the KeePass password manager, many useful options become available for coders who need to manage large numbers secrets.

## Requirements
The Protect-CmsMessage and Unprotect-CmsMessage cmdlets require PowerShell 5.0 or later.

Windows 10, Server 2016 and later come with PowerShell 5.0 or later by default, but earlier operating system owners will need to download the latest version of PowerShell from Microsoft.

The Protect-CmsMessage cmdlet requires a public key from a digital certificate, but not just any certificate will work. Here are the requirements for the public key certificate:

* The certificate must include the "Data Encipherment" or "Key Encipherment" Key Usage in the property details of the certificate.

* The certificate must include the "Document Encryption" Enhanced Key Usage (EKU), which is identified by OID number 1.3.6.1.4.1.311.80.1.

To confirm that your intended certificate meets these requirements, double-click the certificate file (.cer) to view its properties, go to the Details tab, and confirm that the "Enhanced Key Usage" property at least includes this line:

* Document Encryption (1.3.6.1.4.1.311.80.1)

And also in that certificate's properties, confirm that the "Key Usage" field includes one or both of these:

* Key Encipherment
* Data Encipherment

If your certificate is already imported into your local profile, then run MMC.EXE, add the Certificates snap-in (File menu) and double-click the desired certificate in your list of Personal certificates. You can also right-click that certificate > All Tasks > Export, in order to export the certificate to a file (.cer), but do not export the private key associated with this certificate.

If you are using a Windows PKI to obtain certificates, the template used must be configured correctly for CMS encryption. In the certificate template, you must ensure that:

* On the Request Handling tab, you have selected "Encryption" or "Signature and Encryption", and

* On the Extensions tab, you have selected Application Policies and then have added "Document Encryption" to the list of Application Policies.

(Note: If you are not familiar with Windows PKI management, search Microsoft's web site or attend the SANS six-day course SEC505, which includes a full day on Windows PKI. You can also create your own self-signed certificate, which is easier for getting started or testing, but is usually not recommended for production use.)

Officially, the data to be encrypted must be smaller than 2GB, which implies you can encrypt up to 2GB in a single CMS message, but see below for issues/errors.

## Specifying the Certificate to Use
The certificate used by Protect-CmsMessage can be given to the cmdlet in various ways with the "-To" parameter:

* Path to an exported certificate file (.cer)

* Path to a directory containing an exported certificate file (.cer)

* Thumbprint hash value of the certificate in the user's Cert:\ drive

* The certificate's subject name, to be found in the user's Cert:\ drive

* A certificate object stored in a variable

The most intuitive way to do it is to export your certificate to a file (.cer) and provide the full explicit path.

If you provide the path to a folder, the Protect-CmsMessage cmdlet will search that folder for certificate files, select the first one it finds, and try that one. If this fails for any reason, like if the certificate does not meet the EKU requirements, then you'll get a terminating error and no other certificates from that folder will be attempted. There does not appear to be any way to specify the order or give preference to one certificate over another in the folder, such as key size or expiration date, other than with your own code, but, in this case, you end up giving the full path to the desired certificate file anyway.

Using a thumbprint precisely specifies exactly one certificate, which can be good, but it also means you must update that hash value in your code whenever the desired certificate is changed. If you search by subject name using a wildcard, you might get back an array of matching certificates, so take care to choose the correct one. For these reasons, it is often simplest to export the desired certificate to a file, use a standardized file name, and give the explicit path to this file.

For an example of using a certificate stored in a variable in memory, see below. This is convenient when you want to embed the desired certificate inside your script, hence, there is no external certificate file or path to worry about, i.e., the script is a one-file "package" so to speak. If such a script were digitally signed, it helps to ensure integrity and that only the intended public key certificate will be used by Protect-CmsMessage.

## Example Code
Here is some example code to get started. This assumes you already have an exported certificate that meets the above requirements and that the associated private key with that certificate is in your local profile.

The output of each encryption operation will be Base64 text, similar to the following:

```
-----BEGIN CMS-----
MIIB4AYJKoZIhvcNAQcDoIIBwTCCAb0CAQAxggF4MIIBdAIBADBcMEUxFTATBgoJkiaJk/IsZAEZ
FgVsb2NhbDEXMBUGCgmSJomT8ixkARkWB4Rlc3RpbmcxEzARBgNVBAMTClRlc3RpbmctQ0ECE1sA
AACI573u9cj1hfAAAAAAAIgwDQYJKoZIhvcNAQEHMAAEggEAeQ8el2gKzvP9p+g681LaBAICWB98
Tf8t7Mth7ml3fD2mwAaI4NH4m5NXDAJTpJn22axBRbqOYZI4XovIAYgX0fSH8gt0sl5aOwGMt61W
v3VFLdTr9hTg4vCd6NVut1R5G54odbk8D9V92gH0Fys9s3tNX22TnREkrKeqZsfowdoxeId3Jkwn
kg/ONt8ZOSCZKQIeD57IX8UTXmK+D8Q8GuO1cFLwYjZmnlYdXQZ4SDNqAi/XSZAkvVEUChJAD6Nm
1SxAjm0Fy89NBrqSfvX6vy/55v+J11hzYGVlm1be7u2CyjmdQ05r3tV2Q2iwpViDYaknP1P5muEJ
YmPaAhVSODA8BgkqhkiG9w0BBwEwHQYJYIZIAWUDBAEqBBBKkzVjYTsYW0Aq2d4EgwAIgBDUCb3B
IIvMCLR1lmmK2hAs
-----END CMS-----
```

The above ciphertext can be captured to a variable, saved directly to file, posted with Invoke-WebRequest, e-mailed with Send-MailMessage, etc.

```powershell 
# To encrypt the contents of a variable using an exported certificate:
$x = get-process            #Some object data to protect
Protect-CmsMessage -To .\ExportedCert.cer -Content $x -OutFile EncryptedVar.cms
 
# To encrypt a TEXT file using an exported certificate, saving the output to a new file:
Protect-CmsMessage -To .\ExportedCert.cer -Path .\SecretDocument.txt -OutFile EncryptedDoc.cms
 
# To encrypt a variable or TEXT file using an exported certificate, capturing the output to a new variable:
$CipherText1 = Protect-CmsMessage -To .\ExportedCert.cer -Content $x
$ciphertext2 = Protect-CmsMessage -To .\ExportedCert.cer -Path .\SecretDocument.txt
 
# To decrypt CMS ciphertext, assuming you have the private key:
$plaintext = Unprotect-CmsMessage -Path .\EncryptedVar.cms
$plaintext = Unprotect-CmsMessage -Content $CipherText1
$plaintext = Unprotect-CmsMessage -Path .\EncryptedFile.cms
```

## Not A Property Bag
Importantly, when an object is encrypted, it is only the simple ToString() representation of that object which gets encrypted, not the whole "[property bag](http://blogs.msdn.com/b/powershell/archive/2010/01/07/how-objects-are-sent-to-and-from-remote-sessions.aspx)" of the object; hence, it is not (usually) possible to recreate the original object in memory again like with Import-Csv or Import-CliXml.

If you want to reflate an array of objects in memory after decryption with Unprotect-CmsMessage, first convert the array to CSV or XML string data, encrypt with Protect-CmsMessage, decrypt with Unprotect-CmsMessage, then reflate the objects from the plaintext CSV or XML data. To call an object a "property bag" means that it lacks methods, but it also implies that the object can be serialized and then recreated again. In that sense, Protect-CmsMessage does not serialize objects or property bags, and Unprotect-CmsMessage does not reflate, recreate or deserialize the original object or property bag after decryption.

In general, do not expect the (Un)Protect-CmsMessage cmdlets to work reliably with anything other than textual data.

## Encrypting Binary Files
What about encrypting and decrypting binary files or binary data then?

The following commands DO NOT WORK, they fail to restore the original file:

```powershell
# This does NOT work with binary or EXE files:
Protect-CmsMessage -To .\ExportedCert.cer -Path .\InputFile.exe -OutFile OutputFile.cms
Unprotect-CmsMessage -Path .\OutputFile.cms | Set-Content -Path RestoredFile.exe
 
# When the original and restored files are hashed, the hashes will NOT match:
Get-FileHash -Path .\InputFile.exe,.\RestoredFile.exe
```

Nor can you make the above commands work by specifying "Set-Content -Encoding Byte": this raises an error because the output of Unprotect-CmsMessage is text, not an array of bytes.

But if you Base64-encode the binary data first, which converts the raw bytes to a textual representation, then that text can be encrypted with Protect-CmsMessage.

```powershell
# A couple helper functions to go to/from Base64 and binary:
function Convert-FromBinaryFileToBase64
{
[CmdletBinding()]
Param
(
[Parameter(Mandatory = $True, Position = 0, ValueFromPipeline = $True)] $Path,
[Switch] $InsertLineBreaks
)
 
if ($InsertLineBreaks){ $option = [System.Base64FormattingOptions]::InsertLineBreaks }
else { $option = [System.Base64FormattingOptions]::None }
 
[System.Convert]::ToBase64String( $(Get-Content -ReadCount 0 -Encoding Byte -Path $Path) , $option )
}
 
  
 
function Convert-FromBase64ToBinaryFile
{
[CmdletBinding()]
Param( [Parameter(Mandatory = $True, Position = 0, ValueFromPipeline = $True)] $String ,
[Parameter(Mandatory = $True, Position = 1, ValueFromPipeline = $False)] $OutputFilePath )
 
[System.Convert]::FromBase64String( $String ) | Set-Content -OutputFilePath $Path -Encoding Byte
 
}
 
# Convert the binary file to Base64 strings, then encrypt:
dir .\InputFile.exe | Convert-FromBinaryFileToBase64 | Protect-CmsMessage -To .\ExportedCert.cer -OutFile OutputFile.cms
 
# Then the file can be decrypted and the Base64 converted back into a binary file again:
Unprotect-CmsMessage -Path .\OutputFile.cms | Convert-FromBase64ToBinaryFile -OutputFilePath .\RestoredFile.exe
 
# Now the hashes will match:
Get-FileHash -Path .\InputFile.exe,.\RestoredFile.exe
```

Unfortunately, the performance of the above is horrible. A 20MB CMS file, for example, will require more than 10 minutes to decrypt on an Intel Core i7 4790 @ 3.60GHz. Maybe I'm making an obvious mistake in my code, but there are other issues too.

## Limitations, Performance and Error Messages
The most important limitation is that Protect-CmsMessage should only be used to encrypt text or text files, nothing else.

When encrypting objects as text, if you'll need to recreate the original objects or property bags again later, convert the in-memory objects to XML or CSV first, then encrypt that XML/CSV text instead.

If you need to encrypt raw byte arrays or arbitrary binary data, encode that data in Base64, then encrypt the Base64 text.

What is the maximum amount of text that can be encrypted? If you attempt to encrypt more than 2GB of data, the error message is:

```
Protect-CmsMessage : The file is too long. This operation is currently limited to 
supporting files less than 2 gigabytes in size.
```

So the official limit is 2GB, which is fine, nothing can handle infinitely-large files (even 7-Zip can't handle files larger than...ahem...16,000,000,000 GB), but if you attempt to encrypt even 1GB of data, you'll get a different error:

```
Protect-CmsMessage : Exception of type 'System.OutOfMemoryException' was thrown.
```

The above error was produced on a machine with 32GB of memory, a 4GB paging file, a freshly relaunched PowerShell ISE, having run [GC]::Collect() on a fully patched Windows 10 Pro (Aug'2015). So what is the actual maximum file size limit then?

If you encrypt a 200MB text file, it works, but when attempting to decrypt that CMS output file, another error appears:

```
Unprotect-CmsMessage : ASN1 value too large.
```

At this point, I stopped the size testing, the cmdlets are not reliable at anything near the official 2GB limit. At 1/100th the official maximum (20MB) there were no error messages and the hash testing was fine. Perhaps 20MB is the practical limit then? This is just a guess. The next step would be to start playing with the underlying [System.Security.Cryptography.Pkcs.EnvelopedCms](https://msdn.microsoft.com/en-us/library/system.security.cryptography.pkcs.envelopedcms(v=vs.110).aspx) class to whether the limits come from the underlying class or the cmdlet wrappers.

Because of size and performance issues, if you need to quickly encrypt a large amount of data of arbitrary format, I suggest using Protect-CmsMessage to encrypt a 100-character random passphrase and then using that passphrase with 7-Zip to compress and encrypt the data (100 characters * 3 bits of entropy per char = the 7-Zip AES key with some extra entropy cushion). PowerShell 5.0 and later includes the Compress-Archive cmdlet, but this cannot encrypt archives and is not as fast or reliable as 7-Zip with large archives. There are nice 7-Zip modules in the PSGallery too. You could also do a script to create, mount and BitLocker-encrypt a VHD virtual machine image file, then copy the data into that VHD.

(If you wish to use a more pure PowerShell solution, check out Dave Wyatt's excellent [ProtectedData module](https://github.com/dlwyatt).)

## Using Get-CmsMessage
The Get-CmsMessage cmdlet displays metadata about CMS-encrypted contents, but cannot decrypt those contents. You might use this cmdlet when receiving or organizing many CMS messages.

```powershell
# Display metadata about a CMS message, but not decrypt it:
Get-CmsMessage -Path .\little.cms
"test data" | Protect-CmsMessage -To .\ExportedCert.cer | Get-CmsMessage
 
# The output may scroll by for several pages, so this will display just
# some of the metadata and in a more-useful form:
 
# Which certificate was used to encrypt the data:
Get-CmsMessage -Path .\little.cms | Select -ExpandProperty Recipients
(Get-CmsMessage -Path .\little.cms).Recipients
 
# Content type is PKCS #7 because of the historical roots of CMS (see RFC 5652):
Get-CmsMessage -Path .\little.cms | Select -ExpandProperty ContentInfo | Select -ExpandProperty ContentType
(Get-CmsMessage -Path .\little.cms).ContentInfo.ContentType.FriendlyName
 
# Encryption type is AES 256:
Get-CmsMessage -Path .\little.cms | Select -ExpandProperty ContentEncryptionAlgorithm | Select -ExpandProperty Oid
(Get-CmsMessage -Path .\little.cms).ContentEncryptionAlgorithm.Oid.FriendlyName
```

## Certificate Embedded In Script
Instead of reading the recipient's certificate from a file or from your own certificate store in your profile, the certificate can be encoded as hexadecimal text in a byte array and embedded right inside the script. This is handy when you want to use a one-file-only solution and avoid the hassles of managing both a script and separate certificate files.

The downside, of course, is that it becomes more difficult to change that certificate or to use other certificates at different times for different payloads.

This is also a technique to be aware of for the sake of malware forensics or investigating the post-exploitation use of PowerShell on a compromised machine.

See the Protect-CmsMessage_Examples.ps1 script for an example of how to embed a certificate in a script.

Have fun!

## Update History
* 23.Aug.2015: First posted
* 11.Jun.2016: Added 7-Zip notes
* 1.Jun.2017: Moved to GitHub



