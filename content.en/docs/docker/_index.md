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

# Docker server

This section covers the deployement of a standardized [Portainer](https://www.portainer.io/) instance, using [Caddy](https://caddyserver.com/) as reverse proxy, [Authelia](https://www.authelia.com/) as an two-factor and OIDC (OpenID Connect) provider. The root system is the [docker-enabled debian](/docs/cloud-init/#secured-cloud-init-config-with-docker).

## Launch portainer

First create the network that will be used by [caddy-docker-proxy](https://github.com/lucaslorentz/caddy-docker-proxy), if you give it another name than caddy caddy, you'll have to modify the compose files accordingly.

``` sudo docker network create caddy```

Create the following `compose.yml` file. In your registar, create an A record that point to the server's IP, and edit line 14 accordingly.

```yml {hl_lines=[14] style=emacs}
services:
  portainer:
    image: portainer/portainer-ce:sts
    ports:
      - 9443:9443
    volumes:
      - portainer_data:/data
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
    ## The following lines are needed for caddy-docker-proxy.
    networks:
      - caddy
    labels: 
      caddy: portainer.example.com #Change for the domain you'll use.
      caddy.reverse_proxy: "{{upstreams 9000}}"
      ## The following lines are needed for Authelia to provide 2FA.
      caddy.forward_auth.uri: /api/authz/forward-auth
      caddy.forward_auth.copy_headers: "Remote-User Remote-Groups Remote-Name Remote-Email"
      # This next line indicates where to reach Authelia. If you want to limit to a subpath you can modify it as such : caddy.forward_auth: "\subpath app:9091"
      # If accessing Authelia from another server. put the URL (with the https://) of Authelia instead of app:9091 in the following line.
      caddy.forward_auth: app:9091 
      # Uncomment the next if accessing Authelia from another server. header_up is required here as we have Caddy in front of Authelia remotely and we need to let that Caddy know we want to talk to Authelia and not some other site.
      # caddy.foward_auth.header_up: "Host {upstream_hostport}" 
    
networks:
  caddy:
    external: true
    
volumes:
  portainer_data: {}
```

You'll note that most of the lines are caddy related, they won't be used at first but that way everything is there from the start.

Bring portainer up with `sudo docker compose up -d` and it'll be reachable on port 9443 (don't forget to append https://).

## Setting up caddy

Using [caddy-docker-proxy](https://github.com/lucaslorentz/caddy-docker-proxy) which allows for the generation of the caddyfile on the fly by settings labels on the target docking machine.

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
    yourusername: #If you plan on using OIDC, has to be different than the one from Portainer.
        password: hashed_password
        displayname: "Your Displayname"
        email: name@example.com
```

To hash the password, use the command :
```
sudo docker run --rm -it authelia/authelia:latest authelia crypto hash generate argon2
```

Create the ´configuration.yml´ file and paste the [pre-created version](/docs/docker/configuration.yml), editing the lines as specified.

Once all the file are created, time to lock everything up :

```
sudo chown -R root:root /opt/authelia
sudo chmod -R 600 /opt/authelia
```

### Docker-compose for Authelia

Caddy should be already up and running, go to your registar, set the A record for Authelia and create the following compose file. Two values should be edited, the domain on line 25, and the REDIS_PASSWORD on line 42

```yml {hl_lines=[25,42] style=emacs}
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
      X_AUTHELIA_CONFIG_FILTERS: template
      AUTHELIA_IDENTITY_VALIDATION_RESET_PASSWORD_JWT_SECRET_FILE: /config/secrets/JWT_SECRET
      AUTHELIA_SESSION_SECRET_FILE: /config/secrets/SESSION_SECRET
      AUTHELIA_NOTIFIER_SMTP_PASSWORD_FILE: /config/secrets/SMTP_PASSWORD
      AUTHELIA_STORAGE_ENCRYPTION_KEY_FILE: /config/secrets/STORAGE_ENCRYPTION_KEY
      AUTHELIA_STORAGE_POSTGRES_PASSWORD_FILE: /config/secrets/STORAGE_PASSWORD
      AUTHELIA_SESSION_REDIS_PASSWORD_FILE: /config/secrets/REDIS_PASSWORD
# Only uncomment this line if you are using OIDC - if not, Authelia will not start
#      AUTHELIA_IDENTITY_PROVIDERS_OIDC_HMAC_SECRET_FILE: /config/secrets/HMAC_SECRET
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

Authelia should now be reachable via the password you've previously hashed. Confirm this by navigating to the address you've set up for Authelia. Also, try to access Portainer with the address you've set for Portainer, without the port number. Authelia should pop-up and ask for authentification before allowing you to move further.

If everything is working, you can edit the `compose.yml` file for Portainer and comment the port 9443. This will mean that from now on, you have to identify with Authelia to reach Portainer.

## Setting up OIDC for Portainer

Of course, you now have to enter 2 passwords and 1 TOTP to access Portainer, which is, to say the least, tedious. Thus the next and final step is to set up Authelia as an OIDC provider, and set Portainer to use it.

First an additional secret and a 2048 bits RSA key will be needed, so go back to the folder `/opt/authelia/secrets/` and create them :
```
tr -dc A-Za-z0-9 </dev/urandom | head -c 80 | { cat; echo; } | sudo tee HMAC_SECRET
openssl genrsa -out oidc.pem 2048
```

A hashed password will also be needed, so choose a password, and hash it using the following command :

```
sudo docker run --rm -it authelia/authelia:latest authelia crypto hash generate pbkdf2
```

Now, you have to edit the monstruous `configuration.yml` file to add OIDC, [see instructions](/docs/docker/configuration-oidc.yml).

Once the secrets are created and the file modified, you can restart Authelia after uncommenting the `AUTHELIA_IDENTITY_PROVIDERS_OIDC_HMAC_SECRET_FILE` line in the compose file. 

And finally, you have to configure Portainer :

- Visit Settings
- Visit Authentication
- Configure the following options:
  - Authentication Method: OAuth
  - Automatic User Provision: Enable if you want users to automatically be created in Portainer.
  - Provider: Custom
    - Client ID: portainer
    - Client Secret: *THE UNHASHED PASSWORD*
    - Authorization URL: https://auth.example.com/api/oidc/authorization
    - Access Token URL: https://auth.example.com/api/oidc/token
    - Resource URL: https://auth.example.com/api/oidc/userinfo
    - Redirect URL: https://portainer.example.com *NOTE : has to match the redirect_uris: parameter of the configuration.yml file exactly*
    - Logout URL: https://auth.example.com 
    - User Identifier: preferred_username *NOTE : you have to write preferred_username, not write your username*
    - Scopes: openid profile groups email
    - Auth Style: In Params

If you haven't selected Automatic user provision, you have to create a user which matches your authelia username in order to login with Oauth. So if your user in Authelia is called MyUsername, you have to add a user MyUsername in portainer.

And voilà, click the big "OAuth" button while loggin in to Portainer, and it should automagically log you in.
