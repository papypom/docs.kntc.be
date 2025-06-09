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

This section covers the deployement of a standardized portainer instance, using Caddy as reverse proxy. The root system is the [docker-enabled debian](/docs/cloud-init-config/#minimal-cloud-init-config-with-docker).

## Launch portainer

First create the network that will be used by caddy, here I'm naming it caddy.

``` sudo docker network create caddy```

Create a folder called `portainer`, and within create the following `compose.yml` file. Modify as needed.

```yml {hl_lines=[13,17] style=emacs}
services:
  portainer:
    image: portainer/portainer-ce:sts
    ports:
      - 9443:9443
    volumes:
      - portainer_data:/data
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
    networks:
      - caddy
    labels:
      caddy: portainer.example.com
      caddy.reverse_proxy: "{{upstreams 9000}}"
    
networks:
  caddy: #change the network name if needed.
    external: true
    
volumes:
  portainer_data: {}
```

Then, from within the folder, a simple `sudo docker compose up -d` will spin up portainer on port 9443 (don't forget to append https://).

## Setting up caddy

Using [caddy-docker-proxy](https://github.com/lucaslorentz/caddy-docker-proxy) which allows for the generation of the caddyfile on the fly by settings labels on the target docking machine (this is the reason behind the 2 -l in the portainer command).

Replace the content of the highlighted lines as specified.

```yml {hl_lines=[8,16,19] style=emacs}
services:
  caddy:
    image: lucaslorentz/caddy-docker-proxy:ci-alpine
    ports:
      - 80:80
      - 443:443
    environment:
      - CADDY_INGRESS_NETWORKS=caddy #change the network name if needed.
    networks:
      - caddy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - caddy_data:/data
    restart: unless-stopped
    labels:
      caddy.email: EMAIL_ADDRESS_FOR_LETSENCRYPT

networks:
  caddy: #change the network name if needed.
    external: true

volumes:
  caddy_data: {}
```

Now you have access to portainer on HTTPS at the domain specified when created portainer. So far so good. Main issue is now you have a portainer accessible on the internet without MFA, which, in 2025, is a big no-no. So the few next steps will set up Authelia to provide said second factor.

## Setting up Authelia

### Preparing the needed files

You will need : an SMTP account to send email from.

We will create the following directory structure and use bind mounts instead of storing passwords in the configuration, as per [Authelia's documentation](https://www.authelia.com/configuration/methods/secrets/).

```
.
├── configuration.yml
├── secrets
│   ├── JWT_SECRET
│   ├── REDIS_PASSWORD
│   ├── SESSION_SECRET
│   ├── SMTP_PASSWORD
│   ├── STORAGE_ENCRYPTION_KEY
│   └── STORAGE_PASSWORD
└── users_database.yml
```

I'm using the folder `/opt/authelia/`, so first create the needed directories :

```
sudo mkdir -p /opt/authelia/secrets/
```
cd to the newly created folder, and create the needed secrets file :
```
tr -dc A-Za-z0-9 </dev/urandom | head -c 80 | { cat; echo; } | sudo tee JWT_SECRET
tr -dc A-Za-z0-9 </dev/urandom | head -c 80 | { cat; echo; } | sudo tee SESSION_SECRET
tr -dc A-Za-z0-9 </dev/urandom | head -c 80 | { cat; echo; } | sudo tee STORAGE_PASSWORD
tr -dc A-Za-z0-9 </dev/urandom | head -c 80 | { cat; echo; } | sudo tee STORAGE_ENCRYPTION_KEY
tr -dc A-Za-z0-9 </dev/urandom | head -c 80 | { cat; echo; } | sudo tee REDIS_PASSWORD
```

And add a final file with the SMTP password :
```
sudo nano SMTP_PASSWORD
```

Create the `users_database.yml` in the root folder, and copy-paste the following content, editing the highlighted lines : 


```yml {hl_lines=["4-7"] style=emacs}
# User file database https://www.authelia.com/reference/guides/passwords/#yaml-format
# Generate passwords https://www.authelia.com/reference/guides/passwords/#passwords
users:
    yourusername:
        password: hashed_password
        displayname: "Your Displayname"
        email: name@example.com
```

To hash the password, use the command :
```
sudo docker run --rm -it authelia/authelia:latest authelia crypto hash generate argon2
```

Create the ´configuration.yml´ file and paste the [pre-created version](/docs/configuration.yml), editing the lines as specified.

Once all the file are created, time to set everything as write-only :

```
sudo chown -R root:root /opt/authelia
sudo chmod -R 600 /opt/authelia
```

### Docker-compose for Authelia

Caddy should be already up and running, go to your DNS and set the A record for Authelia and create the following compose file. Two values should be edited, the domain on line 22, and the REDIS_PASSWORD on line 39

```yml {hl_lines=[22,39] style=emacs}
name: "authelia"
services:
  app:
    image: authelia/authelia:latest
    restart: unless-stopped
    depends_on:
      - database
      - redis
    volumes:
      - /opt/authelia:/config
    environment:
      AUTHELIA_IDENTITY_VALIDATION_RESET_PASSWORD_JWT_SECRET_FILE: /config/secrets/JWT_SECRET
      AUTHELIA_SESSION_SECRET_FILE: /config/secrets/SESSION_SECRET
      AUTHELIA_NOTIFIER_SMTP_PASSWORD_FILE: /config/secrets/SMTP_PASSWORD
      AUTHELIA_STORAGE_ENCRYPTION_KEY_FILE: /config/secrets/STORAGE_ENCRYPTION_KEY
      AUTHELIA_STORAGE_POSTGRES_PASSWORD_FILE: /config/secrets/STORAGE_PASSWORD
      AUTHELIA_SESSION_REDIS_PASSWORD_FILE: /config/secrets/REDIS_PASSWORD
    networks:
      - caddy
      - authelia
    labels:
      caddy: auth.example.com
      caddy.reverse_proxy: "{{upstreams 9091}}"

  database:
    image: postgres:15
    restart: unless-stopped
    volumes:
      - postgres:/var/lib/postgresql/data
      - /opt/authelia/secrets/STORAGE_PASSWORD:/STORAGE_PASSWORD
    environment:
      POSTGRES_USER: "authelia"
      POSTGRES_PASSWORD_FILE: "/STORAGE_PASSWORD"
    networks:
      - authelia

  redis:
    image: redis:7
    command: "redis-server --save 60 1 --loglevel warning --requirepass EDIT_WITH_THE_REDIS_PASSWORD_CONTENT"
    volumes:
      - redis:/data
    networks:
      - authelia

networks:
  caddy:
    external: true
  authelia:

volumes:
  postgres: {} 
  redis: {}
```

Authelia should now be reachable via the password you've previously hashed. Confirm this by navigating to the address you've set up for Authelia. Once all is working, it is now time to ...

### Setting up authentification for the portainer via Authelia

Add the needed labels to your docker compose file for portainer. We've also removed the port 9443, which is not used anymore.

```yml {hl_lines=[13] style=emacs}
services:
  portainer:
    image: portainer/portainer-ce:sts
#    ports:
#      - 9443:9443
    volumes:
      - portainer_data:/data
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
    networks:
      - caddy
    labels:
      caddy: portainer.example.com
      caddy.reverse_proxy: "{{upstreams 9000}}"
      caddy.forward_auth: app:9091 #Note : if accessing an external authelia, put the URL here
      caddy.forward_auth.uri: /api/authz/forward-auth
      caddy.forward_auth.copy_headers: "Remote-User Remote-Groups Remote-Name Remote-Email"
# Uncomment this line if accessing an external Authelia. header_up is required here as we have Caddy in front of Authelia remotely and we need to let that Caddy know we want to talk to Authelia and not some other site.
#      caddy.foward_auth.header_up: "Host {upstream_hostport}" 
    
networks:
  caddy: #change the network name if needed.
    external: true
    
volumes:
  portainer_data: {}
```

If you want to limit to a subpath you can add it to the line `caddy.forward_auth: app:9091` as such `caddy.forward_auth: "\subpath app:9091"`

## Setting up OIDC for Portainer

Of course, you now have to enter 2 passwords and 1 TOTP to access your portainer, which is, to say the least, tedious. Thus the next and final step is to set up Authelia as an OIDC provider, and allow the config to be digested by Portainer. This is a story for another day.