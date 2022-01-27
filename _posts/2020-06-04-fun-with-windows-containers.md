---
layout: post
title: "Fun with Windows Containers"
date: 2020-06-04
categories: windows, docker
github_comments_issueid: 4
---

This will probably be a rather random list of moans and rants about running Windows containers that'll get updated every time I need to scream or cry.

## Manifest Destiny - how to pick the right tag for your container host

First up Microsoft's rather interesting decision to deprecate "latest" as a tag and then not do a very good job of explaining that you need to know which Windows build your running so you can download an image without getting an error like

```(powershell)
no matching manifest for windows/amd64 10.0.17763 in the manifest list entries
```

or

```(powershell)
Error response from daemon: manifest for mcr.microsoft.com/windows/windows:2004 not found: manifest unknown: manifest tagged by "2004" is not found
```

Just for fun on their [Get started: Run your first Windows container](https://docs.microsoft.com/en-us/virtualization/windowscontainers/quick-start/run-your-first-container) page they provide this helpful tip:

```(text)
If you see an error message that says no matching manifest for unknown in the manifest list entries, make sure Docker isn't configured to run Linux containers.
```

And then give no further clues how you (someone running a Windows container for the first time) might actually solve the issue. I never found the answer so can't help you, sorry. Blame MS. :)

### What's the problem?

Well, it seems that unlike when running Linux where you just pull latest and stuff just works, Windows will only support builds equal to or less than the one you're running. Maybe Linux works this out for you and downloads the most appropriate build? I don't know but it's not an issue I ever ran into before. 

So there's actually a couple of ways to figure out which tag you need to pick from the massive list of tags for each windows image.

```(powershell)
> [System.Environment]::OSVersion.Version

Major  Minor  Build  Revision
-----  -----  -----  --------
10     0      17763  0
```

or the About page in Windows Settings/System

![alt text](https://github.com/ruebroad/ruebroad.github.io/raw/master/images/windows_build_info.png)
