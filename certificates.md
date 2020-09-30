# Certificates

The guide here will walk you through requesting certificates for your instance!
Before you start here you should have already registered the domain name (so it
will resolve). To make life easy we will use nginx on the host instead of trying
to register through the Django application webserver.

## Install dependencies

You should only need to install certbot and nginx once!

```bash
sudo apt-get install -y nginx

# Install certbot (if not already done)
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install -y python-certbot-nginx
```

## Getting New Certificates

After you've bought your domain and edited the DNS records to point to it,
you can define some environment variables that will help to establish you
as the point of contact, etc.

```bash
export EMAIL=<username>@stanford.edu
export DOMAIN=<LABNAME>.stanford.edu
```

You can then start nginx, and check your domain if you want to see if it's directing
there yet.

```bash
sudo service nginx start
```

And then issue this command to get certificates!

```bash
sudo certbot certonly --nginx -d "${DOMAIN}" --email "${EMAIL}" --agree-tos --redirect
```

The prompt is interactive, and will show the locations of certificates

```bash
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator nginx, Installer nginx
Obtaining a new certificate

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/<LABNAME>.stanford.edu/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/<LABNAME>.stanford.edu/privkey.pem
   Your cert will expire on 2020-12-29. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

You can then stop nginx.

```bash
sudo service nginx stop
```

This is probably extra work, but I like the certificates to be easy to find,
so I typically copy just what I need somewhere close to the instance.

```bash
mkdir -p certs
sudo cp /etc/letsencrypt/live/<LABNAME>.stanford.edu/fullchain.pem certs/fullchain.pem
sudo cp /etc/letsencrypt/live/<LABNAME>.stanford.edu/chain.pem certs/chain.pem
sudo cp /etc/letsencrypt/live/<LABNAME>.stanford.edu/privkey.pem certs/domain.key
```

Finally, generate dhparam.pem for extra security.

```bash
openssl dhparam -out certs/dhparam.pem 4096
```

### Updating the docker-compose

At this point, you'll want to create a `docker-compose-override.yml` that 
points to the proxy:

```bash
version: "2.3"

services:
  cvat_proxy:
    environment:
      CVAT_HOST: <LABNAME>.stanford.edu
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./certs/fullchain.pem:/etc/ssl/certs/fullchain.pem:ro
      - ./certs/domain.key:/etc/ssl/private/domain.key:ro
      - ./certs/dhparam.pem:/etc/ssl/certs/dhparam.pem:ro

  cvat:
    environment:
      ALLOWED_HOSTS: '*'
```

Note in the above that we've provided mounts to the certificates that are references in the proxy
configuration, discussed next.

### Updating the nginx configuration

To update the nginx configuration, you'll want to edit `cvat/cvat_proxy/conf.d/cvat.conf.template` to be
the file [cvat.conf.template](cvat.conf.template). This basically adds ssl (port 443) and 
points to the certificates that we've generated above. You can find more detail in the [cvat advanced topics](https://github.com/openvinotoolkit/cvat/blob/master/cvat/apps/documentation/installation.md#advanced-topics) section.


## Renewing Certificates

Wow, 88 days goes by quickly! To renew, you should first stop the nginx container (the webby part)

```bash
docker-compose stop cvat_proxy
```

Next, create a backup of your old certificates.

```bash
# Create recursive backup with date
backup=$(echo /etc/letsencrypt{,.bak.$(date +%s)} | cut -d ' ' -f 2)
sudo cp -R /etc/letsencrypt $backup
```

Start your nginx local server and issue the renew request

```bash
# Start on server and renew!
sudo service nginx start
sudo certbot renew
# Saving debug log to /var/log/letsencrypt/letsencrypt.log
```

Finally, copy updated certs to where we did before.

```bash
sudo cp /etc/letsencrypt/live/<LABNAME>.stanford.edu/fullchain.pem certs/fullchain.pem
sudo cp /etc/letsencrypt/live/<LABNAME>.stanford.edu/chain.pem certs/chain.pem
sudo cp /etc/letsencrypt/live/<LABNAME>.stanford.edu/privkey.pem certs/domain.key
```

And stop nginx and restart nginx container, verify certs at SSL checker online!

```bash
sudo service nginx stop
docker-compose up -f docker-compose.yml -f docker-compose-override.yml -d nginx_proxy
```
