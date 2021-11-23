---
layout: post
header-style: text
title:  "Where are the Strings in Windows Binaries"
subtitle: "TLDR: It's in MUI files"
date:   2019-05-04 00:00:00
author: "David"
tags:
    - Windows
    - Reverse Engineering
---

## Prelude
The Windows Binaries I'm talking about here are the ones that comes default with Windows provided by Microsoft.

## Searching for Strings in the binary
I was analyzing a Windows binary `C:\Windows\System32\where.exe` when I realized the help text of the binary cannot be found anywhere in the binary.

After some futher investigation using `Process Monitor` from  Windows Sysinternals I found out that it is reading from `C:\Windows\System32\en-US\where.exe.mui` during it's execution.

## Searching for Strings in the MUI
I did some googling to find out more about the MUI file type and realized that it's how Windows enable support for different user interface languages.
> Multilingual User Interface (MUI) enables the localization of user interfaces for globalized applications. MUI also supports the creation of resources for any number of user interface languages.

Further googling on how to open it says that it can be opened using 7zip, so I did and found the strings in `.rsrc\string.txt` in the archieve.
```
52	Type "WHERE /?" for usage help.\n
58	ERROR: Invalid directory specified.\n
60	ERROR: "$env:pattern" cannot be used with /R.\n
63	ERROR: Missing pattern in "$env:pattern".\n
64	INFO: Could not find "%s".\n
--------------------------snip--------------------------
```

What's weird is when I ran linux's `strings` on the MUI file which did not manage to find the strings in the MUI file, so I initially thought that the MUI file is compressed.

I then ran `file` on `where.exe.mui` and it returned `PE32 executable (DLL) (GUI) Intel 80386, for MS Windows`, I thought that's weird, so I tried opening the file in CFF Explorer and it worked! So a MUI file is actually a PE file.

If we check the resource directories of `where.exe.mui` using CFF Explorer we can see a resource directory called `String Tables`. If we check the data within the Directory, we can find all the strings we previously seen using 7zip.

The only thing that's weird was each character was followed by a null byte, which turns out to be because the strings are stored as unicode. Now it makes sense why linux's `strings` did not find anything, it does not search for unicode strings. I tried again with the `strings` tool from Windows Sysinternals and it managed to find the strings because it also searches for unicode strings.

I also looked at `bash.exe.mui` and realized that the strings can also be in the `MESSAGETABLE` section of the resource directory, which have the ID of `0xb`.
> All the resource type can be found from the [Microsoft Documentation](https://docs.microsoft.com/en-us/windows/desktop/menurc/resource-types).

## Ending Notes
Most of the Windows binaries does this, but not all, if you want to extract all the strings used by a Windows binary you should combine finding strings in the binary and the MUI. 

The strings can be in `STRING` (ID of `0x6`) and `MESSAGETABLE` (ID of `0xb`) section of the resource directory of the MUI file.

I have made a [tool in rust](https://github.com/PotatoDrug/MUI-Strings) to retrieve the strings from a MUI file.