---
author: "Or Elimelech"
date: 2025-01-15
title: Support Touch ID for sudo
tags: ['macOS', 'setup']
---

Just got a new computer again (new job) and realized I need to reconfigure sudo to work with touch-id.
I then realized I didn't have touch-id enabled for sudo in a long time, the reason is that every OS update it resets the config.
Today when enabling it, I have realized that there's a file called `/etc/pam.d/sudo_local.template` which does exactly that.

`sudo_local` is a filed that will not be overwritten when upgrading between OS versions.

```sh
sudo cp /etc/pam.d/{sudo_local.template,sudo_local}

# Edit the file and uncomment this line!
auth       sufficient     pam_tid.so
```
