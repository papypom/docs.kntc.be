---
title: "Docker Server"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

## Docker server

This section covers the deployement of a standardized portainer instance, using Caddy as reverse proxy.

## Launch portainer

Enter the following command :

```text {hl_lines=[3] style=emacs}
sudo docker run -d -p 9443:9443 --name portainer --restart=always \
    -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data \
    --network=YOUR_NETWORK -l caddy=YOUR_DOMAIN \
    -l "caddy.reverse_proxy={{upstreams 9000}}" \
    -l caddy.tls=internal portainer/portainer-ce:sts 
```
