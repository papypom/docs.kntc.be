---
title: "Hacks"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

## Hacks

This sections covers a bit of everything, each time I had to go off the beaten track to do stuff.

## Set default path in rclone

Usually, I set up [rclone](https://rclone.org/) to use SFTP. It works, but always ask for the path to destination folder, with no way to set a default path. A solution is to add an alias to your config file (default path `~/.config/rclone/rclone.conf`), as shown hereunder.

```text {hl_lines=[4,5,12] style=emacs}
[rclone_sftp]
type = sftp
host = docs.kntc.be
user = YOUR_SSH_USER
pass = HASHED_SSH_PASSWORD
shell_type = unix
md5sum_command = md5sum
sha1sum_command = sha1sum

[docs.kntc.be]
type = alias
remote = rclone_sftp:DEFAULT/PATH/
```

Don't forget to edit the highlighted lines.