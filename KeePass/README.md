# Launch PowerShell Script From Within KeePass And Include Password Secure String Credential

You want to 1) launch PowerShell.exe using the KeePass password manager by double-clicking an entry in KeePass, perhaps to run a script, 2) inject a password stored in KeePass into that PowerShell.exe process as a variable, such as for a secure string or a PSCredential object, but 3) you cannot expose the password as a command-line argument or other plaintext channel when launching PowerShell.exe because it could be logged or captured. This article will show you how to do it.

So, imagine having a folder in KeePass with dozens of entries for PowerShell commands and scripts you regularly run, but without the hassle of being prompted for a password or other secret every time you execute one of the commands. Just double-click the command you want and rely on KeePass to encrypt the password, web token, encryption key or other secret your PowerShell command needs.

## What Is KeePass?
[KeePass](http://keepass.info) is a free, open source password manager utility for Windows, Linux, Mac OS X, Android, iPhone, Blackberry, and other platforms. KeePass can store many types of secrets, not just passwords, including files, images, script code, encryption keys, credit card numbers, and other valuables.

KeePass includes several [security features](http://keepass.info/features.html), has been around for years, and is actively maintained. It is one of the most popular password managers available on the Internet (and the current author's favorite). KeePass can also be scripted with PowerShell.

## It's Not Just For Passwords
You can launch PowerShell from within the KeePass graphical application itself. This is very handy because you don't want to hard code passwords, encryption keys, wireless keys, web authentication tokens, or other secrets into your PowerShell scripts. But all these secrets can be encrypted in the KeePass database though.

The challenge is how to conveniently launch PowerShell and get the password or secret stored in KeePass into that PowerShell host process as a variable. And this must be done without exposing the password when launching PowerShell (since the password may get logged somewhere, or hackers and malware could query the list of running processes to see what arguments were used to launch those processes, or perhaps scrape the contents of the clipboard).

What we need is something like an encrypted channel between the KeePass process and the PowerShell.exe process. How?

Fortunately, KeePass knows how to use the Windows Data Protection API ([DPAPI](https://en.wikipedia.org/wiki/Data_Protection_API)) to encrypt a password stored in its database, encode those encrypted bytes with Base64, then pass that Base64-encoded string into PowerShell.exe using the -Command parameter as a command-line argument.

While DPAPI isn't perfect, it is difficult to defeat when you have a decent passphrase for logging into Windows, especially on Windows 10 Anniversary Update and later (Q3'2016), or when logging on with a smart card.

I know this might sound hard to set up, but it only takes a minute; you just have to paste some code into a KeePass entry, and you don't need to install any KeePass plugins either.

## KeePass Command URL (CMD://)
The KeePass documentation is great, and it describes [how to execute a command](http://keepass.info/help/base/autourl.html#cmdln) from within KeePass; for example, you can [launch](http://keepass.info/help/base/autourl.html#rdpts) a Remote Desktop Protocol (RDP) connection from within KeePass using Microsoft's built-in MSTSC.EXE.

Let's create a KeePass entry to run a PowerShell script which requires a PSCredential object that contains a username and an obfuscated password (normally created with Get-Credential), like when running Get-WmiObject.

**First**, right-click on KeePass and run it as Administrator. This is necessary to launch PowerShell.exe with administrative privileges from within KeePass, and it's good for [UIPI](https://en.wikipedia.org/wiki/User_Interface_Privilege_Isolation) protection of the KeePass process itself too.

**Second**, in KeePass, pull down the Edit menu, select Add Group, and create a new folder in KeePass for keeping your command entries organized. You might call the new group folder "Commands", for example.

**Third**, create a new KeePass entry (Edit menu > Add Entry) with the following fields:

**Title**: Path to the PowerShell script you want to run which requires a PSCredential object as an argument, e.g., "C:\AdminScripts\Script47.ps1 -Credential $creds" (without the quotes).

**User Name**: The authority\username string that will be used to create the PSCredential object, e.g., "Server12\Administrator".

**Password**: The password or other secret to be passed securely into the script.

**UR**L: Paste the following code into the URL field:

```powershell
cmd://powershell.exe -command "{NOTES}"
```

**Notes**: Paste in the following code into the Notes field:

```powershell
$username = '{USERNAME}';

$cipherbytes = [System.Convert]::FromBase64String( '{PASSWORD_ENC}' );

[System.Reflection.Assembly]::LoadWithPartialName('System.Security') | Out-Null;

[byte[]] $m_pbOptEnt = @(0xA5,0x74,0x2E,0xEC);

$plainbytes = [System.Security.Cryptography.ProtectedData]::Unprotect($cipherbytes, $m_pbOptEnt, 0);

$password = [System.Text.Encoding]::UTF8.GetString( $plainbytes );

$secstring = ConvertTo-SecureString -asPlainText -Force -String $password;

$creds = New-Object System.Management.Automation.PSCredential($username, $secstring);

$password = $null; Remove-Variable -Name password;

{TITLE}
```

That's it! Save the entry, then execute the script by either 1) double-clicking the URL column for the entry (you may need to add the URL column using the View menu), 2) selecting the entry and clicking the green 'Open URL' globe icon in the toolbar, or 3) selecting the entry and pressing Ctrl-U.

## How It Works
Before executing your command, KeePass reads the text of the Title, User Name, Password and Notes fields in your entry, then substitutes these strings in for the {TITLE}, {USERNAME}, {PASSWORD_ENC}, and {NOTES} placeholders, which act kind of like environment variables. This occurs before PowerShell.exe sees any of the text.

Notice how the {TITLE} of the KeePass entry is at the bottom of the Notes. This is how the Title field in KeePass becomes the command to be run in PowerShell. In the above example, the PSCredential object with your password is stored in the $creds variable, hence, your script or command to be run should accept the $creds variable as an argument.

Importantly, the {PASSWORD_ENC} placeholder does not contain the plaintext password, but a Base64-encoded array of encrypted bytes. The bytes are encrypted using Windows DPAPI, which means that only you, and only on this computer where you are running KeePass, can decrypt the bytes to recover the plaintext password again. If an adversary reads this Base64 string from a log or scrapes it from memory, the adversary will not be able to decrypt the string without possession of your logon passphrase/hash and other data unique to the computer where you are running KeePass. If an adversary has kernel-mode malware already installed on the box to acquire this information, then none of these DPAPI details matter anymore, the box has been fully compromised, and so has KeePass or any other password manager application at this point.

A full analysis of DPAPI cannot be rehashed in this blog, so please start here to start following the chain of links, but suffice it to say that a major flaw in DPAPI would be a much bigger problem than just an inconvenience for the little KeePass trick being discussed here (the word "catastrophe" comes to mind...).

The $m_pbOptEnt variable in the Notes is not the DPAPI decryption key, by the way, it's just a constant I copied out of the KeePass source code, which anyone can do (look in \KeePassLib\Utility\StrUtil.cs).

Be aware that all the other placeholders besides {PASSWORD_ENC} are passed into PowerShell.exe in plaintext as command-line arguments. The command-line arguments of a process can be logged or potentially read by other users/hackers/malware on the same box as you.

## Misc Tips
If you want to conceal the other placeholder data too, such as the {USERNAME} placeholder, those strings must be embedded into the Password field of the KeePass entry and then parsed out by your script after decryption of the {PASSWORD_ENC} encrypted string; for example, you might delimit the Password field into several subfields using "+::+" or some other character sequence that can be -Split into an array of strings once inside your script (it could even be XML).

If you need the plaintext Password field in PowerShell after it is launched, delete or comment out the following line from the Notes field:

```powershell
# $password = $null; Remove-Variable -Name password;
```

If you need PowerShell.exe to remain open after executing your command, change the URL field in the KeePass entry to include the -NoExit switch:

```powershell
cmd://powershell.exe -noexit -command "{NOTES}"
```

Instead of running a PowerShell script from the drive, you could place the command(s) you want to run into the Title field or the Notes field of the KeePass entry.

If you are getting errors related to quoting or parsing, try experimenting with the -EncodedCommand argument when launching PowerShell.exe, and see the KeePass [{T-CONV:/Text/Base64}](http://keepass.info/help/base/placeholders.html) placeholder too to let KeePass handle the Base64 encoding for you. Adding semicolons at the end of each command in the Notes field also helps to prevent parsing errors (ending semicolons are not normally required in PowerShell).

Updating the passwords in KeePass can be a challenge, especially when resetting the local Administrator account password every night using a PowerShell script on thousands of hosts, so it's fortunate that KeePass entries can be updated by PowerShell script too.

There are other ways to handle encrypted secrets in PowerShell too, such as using the Protect-CmsMessage cmdlet.

The other fields of a KeePass entry can also be passed into PowerShell.exe as variables (see the KeePass documentation on the [placeholders](http://keepass.info/help/base/placeholders.html)). For example, on the Advanced tab of your KeePass entry, you could add two custom fields named "Value001" and "Value002", then access those fields as variables in your PowerShell code; just change the Notes field of the KeePass entry to look like this:

```powershell
$username = '{USERNAME}';
$pw = '{PASSWORD_ENC}';
$title = '{TITLE}';
$value1 = '{S:Value001}';
$value2 = '{S:Value002}';
```

**Warning!** Just remember that everything other than the {PASSWORD_ENC} placeholder is given in plaintext as arguments to PowerShell.exe when it is launched, which could be logged or captured, so do not pass in any secrets this way. This is especially a problem when running PowerShell with the -NoExit switch and then leaving PowerShell running for hours or days. Secrets from KeePass can also be recorded in the Windows Event Logs or in the PowerShell [transcription logs](https://blogs.msdn.microsoft.com/powershell/2015/06/09/powershell-the-blue-team/) too. It's best to assume everything in KeePass sent to PowerShell other than {PASSWORD_ENC} can be read in plaintext by your adversaries.

Note that even if you do "$password = $null" and force a Gen2 garbage collection, the plaintext password string will still be in memory, so it's best to avoid the -NoExit switch when launching PowerShell.exe with KeePass (but this is a minor issue, your normal PowerShell process is open all day with other secrets in memory too).

As a future project, it would be nice to have a KeePass plugin to right-click an entry and run a PowerShell command so that 1) all the fields would be encrypted with DPAPI, not just the password field, and 2) the data could be optionally fed to PowerShell via a named pipe or other in-memory IPC mechanism instead of a command-line argument. Even better would be to have this built into KeePass by default, not as a plugin. (Incidentally, if you want to see how PowerShell can use named pipes, look in the BinaryData folder.)

## Clipboard Trick
There is another way to get a password out of KeePass and into a PowerShell script, but it's not as secure or reliable. Right-click the KeePass entry and select Copy Password. The password is now in the clipboard. Run your PowerShell script somehow (through KeePass or just execute the script like normal in the shell) and near the top of your script have commands like:

```powershell
$Password = Get-Clipboard
1..50 | ForEach { Set-Clipboard -Value 'AAA' }
```

This requires PowerShell 5.0 or later. You also have to edit the script to make it dependent on this clipboard trick (not very pretty). The "Set-Clipboard" part will clear the clipboard afterwards, but it has to do it 50+ times just in case a [clipboard manager](https://en.wikipedia.org/wiki/Clipboard_manager) is installed. Even this can't prevent malware from snagging the password from the clipboard anyway, but it doesn't hurt to try. In KeePass, you can control how long before KeePass itself purges the clipboard by pulling down the Tools menu > Options > Security tab.

## KeePass Security Best Practices
For a list of security best practices for KeePass, see [this article](https://cyber-defense.sans.org/blog/2015/08/13/powershell-for-keepass-sample-script), which also gives sample PowerShell code for scripting the management of KeePass.

Of course, KeePass and PowerShell are only as secure as the host computer on which they are running. For a SANS Institute training course on Windows security, please consider attending my six-day "[Securing Windows and PowerShell Automation (SEC505)](https://sans.org/sec505)" course online or at a SANS conference. Hope to see you there! — @JasonFossen

## Update History

* 25.May.2016 - first posted.

* 3.Jun.2016 - added silly clipboard trick.

* 1.Jun.2017 -- moved to GitHub.

