+++
title = 'Publish Website'
date = 2025-05-31T23:02:56+02:00
draft = false
+++

## How to publish using current config

Rclone is set to have a remote called 'docs.kntc.be' :

```
hugo --gc --minify
rclone sync --interactive public/ docs.kntc.be:
```