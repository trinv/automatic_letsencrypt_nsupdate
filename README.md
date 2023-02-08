## AUTOMATIC LETSENCRYPT WILDCARD CERT RENEWAL WITH NSUPDATE

I’m using a wildcard cert from letsencrypt. Currently there is only one way how to verify that you hold the domain you are requesting cert for: creating TXT record in that domain. You can do it by hand when asked by certbot but you don’t want to do this each 90 days. If you are running your own DNS servers with PowerDNS like me there’s an elegant solution: RFC2136. This allows you to update your zones without writing config files and restarting anything.

Let’s see how to do this:

First check in ```/etc/powerdns/pdns.conf``` that you have DNS updates allowed and where from. You can also add another host if you are are requesting your certs from elsewhere. This is default:
```
#################################
# allow-dnsupdate-from A global setting to allow DNS updates from these IP ranges.
#
# allow-dnsupdate-from=127.0.0.0/8,::1
```
Next, create a hook script with following content and make it executable:
```
$ cat <<\EOF  > /opt/letsencrypt-dns-hook.sh 
#!/bin/bash

CREATE_DOMAIN="_acme-challenge.$CERTBOT_DOMAIN"

echo "
server 127.0.0.1
update delete $CREATE_DOMAIN TXT
update add $CREATE_DOMAIN 60 TXT $CERTBOT_VALIDATION
send
" | nsupdate

sleep 10
EOF
$ chmod +x /opt/letsencrypt-dns-hook.sh
```
This script takes variables passed in by certbot and creates a nsupdate request to DNS which is then executed (make sure you have nsupdate installed). I also added 10 second delay to allow for the change to propagate to my secondary DNS server.

Then it’s only a matter of using this script with certbot like this:
```
# /opt/certbot-auto certonly --manual --preferred-challenges=dns --manual-auth-hook /opt/letsencrypt-dns-hook.sh -d *.danman.eu
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator manual, Installer None
Cert is due for renewal, auto-renewing...
Renewing an existing certificate
Performing the following challenges:
dns-01 challenge for danman.eu

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
NOTE: The IP of this machine will be publicly logged as having requested this
certificate. If you're running certbot in manual mode on a machine that is not
your server, please ensure you're okay with that.

Are you OK with your IP being logged?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: y
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
- Congratulations! Your certificate and chain have been saved at:
/etc/letsencrypt/live/danman.eu/fullchain.pem
Your key file has been saved at:
/etc/letsencrypt/live/danman.eu/privkey.pem
Your cert will expire on 2019-01-17. To obtain a new or tweaked
version of this certificate in the future, simply run certbot-auto
again. To non-interactively renew *all* of your certificates, run
"certbot-auto renew"
- If you like Certbot, please consider supporting our work by:

Donating to ISRG / Let's Encrypt: https://letsencrypt.org/donate
Donating to EFF: https://eff.org/donate-le
```
I hope this post will help someone. Feel free to comment and share.

Bye!
