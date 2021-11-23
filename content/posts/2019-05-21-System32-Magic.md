---
title:  "System32 Magic"
subtitle: "System32, SysWOW64 and Sysnative"
date:   2019-05-21 00:00:00
author: "David"
tags:
    - Windows
---

So I was using **32-bit** python on windows and trying to read a binary in System32.

```python
with open(r'C:\Windows\System32\FileHistory.exe', 'rb') as f:
    content = f.read()
```

Then I was thrown the following error.

![Ah shit, here we go again](/2019-05-21-System32-Magic/error.jpg)

This makes no sense, I know the file exists in System32.

After some hair tearing and swearing, I figured it out. This error occurs because 32-bit applications are redirected to SysWOW64 when they try to access System32 and there is no `FileHistory.exe` in SysWOW64.

To access the real System32 with 32-bit applications, replace System32 with Sysnative. Sysnative is a special alias that is only visible and accessible from 32-bit programs. So in this case I have to use the following path to actually read the file.

```python
with open(r'C:\Windows\Sysnative\FileHistory.exe', 'rb') as f:
    content = f.read()
```

## Why?

Microsoft wants to split the DLLs and other stuff used by 64-bit and 32-bit applications. 64-bit DLLs will be located in System32 because it is a hardcoded path by a lot of apps.

Intuitively SysWOW64 seems like it should contain 64-bit stuff, but WOW64 stands for *Windows 32-bit on Windows 64-bit* so it actually contains 32-bit stuff.
> In computing on Microsoft platforms, WoW64 (Windows 32-bit on Windows 64-bit) is a subsystem of the Windows operating system capable of running 32-bit applications on 64-bit Windows.

Why not keep 32-bit stuff in System32 and apply the redirection to 64-bit apps instead, and name the 64-bit folder something more intuitive like System64 so it won't be so confusing? Maybe Microsoft is in the forefront of implementing security by confusion.

## Resources
- [The 'Sysnative' folder in 64-bit Windows explained](https://www.samlogic.net/articles/sysnative-folder-64-bit-windows.htm)
- [Difference between System32 and SysWOW64 folders in Windows 10](https://www.thewindowsclub.com/difference-system32-and-syswow64-folders)
- [WOW64](https://en.wikipedia.org/wiki/WoW64)
