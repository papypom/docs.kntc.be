---
title: "NextCloud "
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Nextcloud Backend

Using our standardized [Debian server](/docs/cloud-init/) and [Docker stack](/docs/docker/) (Docker, Caddy, Portainer, Authelia), we will publish the needed containers for a Nextcloud backend, this includes Collabora for document editing, the high-performance backend for Talk and the Whiteboard.

## Collabora

Setting up Collabora is really simple, simple cut and paste this compose file in your Portainer, modify the two highlighted lines, the first one is to set the domain that is allowed to reach your Collabora, the other one the domain from which your Collabora will be reachable (don't forget to create the needed A record).

If you need more than one domain, simply add aliasgroup2, aliasgroup3, ...

```yml {hl_lines=[6,14] style=emacs}
services:
  collabora:
    image: collabora/code
    restart: unless-stopped
    environment:
      - aliasgroup1=https://<domain1>:443
      - dictionaries=en_US fr_FR nl de_DE #Add whatever suits your needs
      - TZ=Europe/Berlin #Change if needed
      - extra_params=
       --o:ssl.enable=false
       --o:ssl.termination=true
       --o:logging.level=warning
       --o:logging.disable_server_audit=true
    volumes:
      - /usr/share/fonts/truetype/:/opt/collaboraoffice/share/fonts/truetype/local/:ro
    cap_add:
     - MKNOD
    networks:
      - caddy
    labels:
      caddy: collabora.example.com
      caddy.reverse_proxy: "{{upstreams 9980}}"

networks:
  caddy:
    external: true
```

If you need additional fonts (a quick Github search should provide the needed files), paste them in the `/usr/share/fonts/truetype/` folder of your server. Don't forget to `chown root:root` them.

Then go to your Nextcloud Administration, go to Office, and set the domain of the freshly created Collabora and add the IP of your server in the WOPI list.

## High-performance Backend

I've tried to install Janus, Turn, Co-Turn, to set everything right, the works. The easiest way by far is to use the HPB image which has all the needed components in one. So, once again, copy-paste the following compose file.

TBD.