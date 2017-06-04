# Manipulate Arrays of Bytes

PowerShell can manipulate and convert binary byte arrays, which is important for malware analysis, interacting with TCP ports, parsing binary log data, and a myriad other tasks. This is a collection of PowerShell functions and notes for manipulating byte data. All the code here is in the public domain. If you find an error or have a suggested addition, please post a comment! The intention is to create a one-stop shop for PowerShell functions related to byte arrays and binary file handling.

## Background: Byte Array Issues
Handling byte arrays can be a challenge for many reasons:

* Bytes can be arranged in big-endian or little-endian format, and the [endianness](http://en.wikipedia.org/wiki/Endianness) may need to be switched by one's code on the fly, e.g., Intel x86 processors use little-endian format internally, but TCP/IP is big-endian.

* A raw byte can be represented in one's code as a .NET object of type System.Byte, as a hexadecimal string, or in some other format, and this format may need to be changed as the bytes are saved to a file, passed in as an argument to a function, or sent to a TCP port over the network.

* Hex strings can be delimited in different ways in text files ("0xA5,0xB6" vs. "0xA5:0xB6" vs. "A5-B6") or not delimited at all ("A5B6").

* Some cmdlets inject unwanted newlines into byte streams when piping.

* The redirection operators (> and >>) mangle byte streams as they attempt on-the-fly Unicode conversion.

* Bytes which represent text strings can encode those strings using ASCII, UTF8+BOM, UTF16, UTF16-BE, etc.
Newline delimiters in text files can be one or more different bytes depending on the application and operating system which created the file.

* Some .NET classes have unexpected working directories when their methods are invoked, so paths must be resolved explicitly first.

* StdIn and StdOut in PowerShell on Windows are not the same as in other languages on other platforms, which can lead to undesired surprises.

* Manipulating very large arrays can lead to performance problems if the arrays are mishandled, e.g., not using a [Ref] where appropriate, constantly recopying to new arrays under the hood, recasting to different types unnecessarily, using the wrong .NET class or cmdlet, etc.

## System.Byte
All variables in PowerShell are .NET objects, including 8-bit unsigned integer bytes. A byte object is an object of type [System.Byte](http://msdn.microsoft.com/en-us/library/system.byte%28VS.80%29.aspx) in the .NET class library, hence, it has properties and methods accessible to PowerShell (you can pipe them into get-member). To create a single Byte object or an array of Bytes:

```powershell
[Byte] $x = 0x4D
[Byte[]] $y = 0x4D,0x5A,0x90,0x00,0x03
```

If you don't cast as a [Byte] or [Byte[]] array, then you'll accidentally get integers. When in doubt, check the type with get-member.

If you want your array to span multiple lines and include comments, that's OK with PowerShell, and remember that you can simply paste the code into your shell, you don't always have to save it to a script file first, hence, you can simply copy the following code and paste it into your shell, it'll work as-is:

```powershell
[Byte[]] $payload = 0x00,0x00,0x00,0x90,   # NetBIOS Session
0xff,0x53,0x4d,0x42,                       # Server Component: SMB
0x72,                                      # SMB Command: Negotiate Protocol
0x00,0x00,0x00,0x00                        # NT Status: STATUS_SUCCESS
```

## Bitwise Operators (AND, OR, XOR)
PowerShell includes support for some bit-level operators that can work with Byte objects: binary AND (-band), binary OR (-bor) and binary XOR (-bxor). See the [about_Comparison_Operators](http://technet.microsoft.com/en-us/library/dd315321.aspx) help file for details, and also see the [format operator](http://msdn.microsoft.com/en-us/library/txafckwd.aspx) (-f) for handling hex and other formats. PowerShell 2.0 and later supports bitwise operators on 64-bit integers too.

## Bit Shifting (<<, >>)
PowerShell 3.0 includes two operators for [bit shifting](http://en.wikipedia.org/wiki/Bitwise_operation): -shl (shift bits left) and -shr (shift bits right with sign preservation). You can also get bit shifting functions at [Joel Bennett's site](http://huddledmasses.org/powershell-needs-shift-operators/).

## Get/Set/Add-Content Cmdlets
The following PowerShell cmdlets will take an "-encoding byte" argument, which you will want to use whenever manipulating raw bytes with cmdlets in order to avoid unwanted on-the-fly Unicode conversions:

* Get-Content
* Set-Content
* Add-Content

The Out-File cmdlet has an -Encoding parameter too, but it will not take "byte" as an argument. The redirection operators (">" and ">>") actually use Out-File under the hood, which is why you should not use redirection operators (or Out-File) when handling raw bytes. The problem is that PowerShell often tries to be helpful by chomping and converting piped string data into lines of Unicode, but this is not helpful when you wish to handle a raw array of bytes as is.

To read the bytes of a file into an array:

```powershell
[byte[]] $x = get-content -encoding byte -path file.exe
```

To read only the first 1000 bytes of a file into an array:

```powershell
[byte[]] $x = get-content -encoding byte -path file.exe -totalcount 1000
```

However, the performance of get-content is not great, even when you set "-readcount 0". The get-filebyte function is faster.

The same is not true, though, for set-content and add-content; their performance is probably adequate for even 5MB of data, but the write-filebyte function below is still much faster when working with more than 5MB.

Remember that you can't pipe just anything into set-content or add-content when using byte encoding, you must pipe objects of type System.Byte specifically. You will often need to convert your input data into a System.Byte[] array first (and there are functions below for this).

To overwrite or create a file with raw bytes, avoiding any hidden string conversion, where $x is a Byte[] array:

```powershell
set-content -value $x -encoding byte -path outfile.exe
```

To append to or create a file with raw bytes, avoiding any hidden string conversion, where $x is a Byte[] array:

```powershell
add-content -value $x -encoding byte -path outfile.exe
```

## Toggle Big/Little Endian (With Sub-Widths)
Different platforms, languages, protocols and file formats may represent data in big-endian, little-endian, middle-endian or some other format (this is also called the "[NUXI problem](http://betterexplained.com/articles/understanding-big-and-little-endian-byte-order/)" or the "[byte order problem](http://3bc.bertrand-blanc.com/endianness05.pdf)"). If the relevant unit of data within an array is the single byte, then reversing the order of the array is sufficient to toggle [endianness](http://en.wikipedia.org/wiki/Endianness), but if the unit to be swapped is two or more bytes (within a larger array) then a simple reversing might not be desired because then the ordering is changed within that unit too. Ideally, a single function could be called multiple times, if necessary, with different unit lengths on a chopped up array of bytes to achieve the right endianness both within and across units in the original array. However, usually you'll probably have just an array of bytes (1 unit = 1 byte) that simply needs to be reversed to toggle the endianness.

## Misc Notes
The "0xFF,0xFE" bytes at the beginning of a [Unicode](http://en.wikipedia.org/wiki/Unicode) text file are [byte order marks](http://msdn.microsoft.com/en-us/library/dd374101%28VS.85%29.aspx) to indicate the use of little-endian UTF-16.

"0x0D" and "0x0A" are the ASCII carriage-return and linefeed ASCII bytes, respectively, which together represent a Windows-style [newline](http://en.wikipedia.org/wiki/Newline) delimiter. This Windows-style newline delimiter in Unicode is "0x00,0x0D,0x00,0x0A". But in Unix-like systems, the ASCII newline is just "0x0A", and older Macs use "0x0D", so you will see these formats too; but be aware that many cmdlets will do on-the-fly conversion to Windows-style newlines (and possibly Unicode conversion too) when saving back to disk. When hashing text files, be aware of how the newlines and encoding (ASCII, UTF8-BOM, UTF16, UTF16-BE, etc) may have changed, since "the same" text will hash to different thumbprints if the newlines or encoding have unexpectedly changed.

## Manipulate-Binary.ps1 Script
For more functions, see the Manipulate-Binary.ps1 script and the other scripts in this folder.

## Legal
Public domain, provided "AS IS" without warranties or guarantees of any kind.

## Update History
* 11.Feb.2010: First posted to SANS.
* 1.Jun.2017: Moved to GitHub.

