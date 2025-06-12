---
title: "Tips"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Tips

This sections covers things I don't do often, but I'm happy to know they exist.

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

## Nano

I use nano as a text editor. Sue me. When editing, if you want to keep the line/character number on screen, add the `-c` flag, and you can open a file at a given line with the `+line_nbr` flag. To open the file bob.yml at the line 100, with line numbering activated, the command would be `nano -c +100 bob.yml`.

## 