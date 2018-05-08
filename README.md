# acme-tiny-dns

This project is a fork of [acme-tiny](https://github.com/diafygi/acme-tiny).
Thanks a log to [diafygi](https://github.com/diafygi) to let us escape from
all the other ACME client craziness.

The rest of this README is only going to be about stuff specific to this code.

## How to use this script

If you already have a Let's Encrypt issued certificate and just want to renew,
you should only have to do Steps 3 and 6.

### Step 1: Create a Let's Encrypt account private key (if you haven't already)

You must have a public key registered with Let's Encrypt and sign your requests
with the corresponding private key. If you don't understand what I just said,
this script likely isn't for you! Please use the official Let's Encrypt
[client](https://github.com/letsencrypt/letsencrypt).
To accomplish this you need to initially create a key, that can be used by
acme-tiny, to register an account for you and sign all following requests.

```
openssl genrsa 4096 > account.key
```

#### Use existing Let's Encrypt key

Alternatively you can convert your key, previously generated by the original
Let's Encrypt client.

Go check [acme-tiny README section](https://github.com/diafygi/acme-tiny#use-existing-lets-encrypt-key).

### Step 2: Create a certificate signing request (CSR) for your domains.

The ACME protocol (what Let's Encrypt uses) requires a CSR file to be submitted
to it, even for renewals. You can use the same CSR for multiple renewals. NOTE:
you can't use your account private key as your domain private key!

```
domain="example.com"
# Generate a domain private key (if you haven't already)
openssl genrsa 4096 > "$domain.key"

# For a single domain
openssl req -new -sha256 -key "$domain.key" -subj "/CN=yoursite.com" > "$domain.csr"

# For multiple domains (use this one if you want both www.example.com and example.com)
openssl req -new -sha256 -key "$domain.key" -subj "/" -reqexts SAN -config <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:$domain,DNS:www.$domain")) > "$domain.csr"
```

### Step 3: Write a hook script that will talk to your DNS server and update the zone

DNS-01 protocol requires the Letsencrypt challenge to be stored in a DNS record of type TXT
and name `_acme-challenge` in your domain's zone.

How to do this will depend on each DNS implementation, and you have to provide your own script.
This script will be called by `acme_tiny_dns.py`:

```
# Add the record
subprocess.check_call([hook, 'update', domain, record_value])
# Remove the record
subprocess.check_call([hook, 'cleanup', domain])
```

so have your hook script do the things (add the record, delete the record).

One example can be found using Gandi and Ruby [here](https://github.com/conchyliculture/gandy).


### Step 4: Get a signed certificate!

Now that you have setup your server and generated all the needed files, run this
script on your server with the permissions needed to write to the above folder
and read your private account key and CSR.

```
# Run the script on your server
python acme_tiny_dns.py --account-key ./account.key --csr "$domain.csr" --domain "${domain}" --hook /path/to/hook/script > ./signed_chain.crt
```

### Step 5: Install the certificate

You're on your own.

Some help in [acme-tiny README](https://github.com/diafygi/acme-tiny#step-5-install-the-certificate)

### Step 6: Setup an auto-renew cronjob

Congrats! Your website is now using https! Unfortunately, Let's Encrypt
certificates only last for 90 days, so you need to renew them often. No worries!
It's automated! Just make a bash script and add it to your crontab (see below
for example script).

Example of a `renew_cert.sh`:
```sh
#!/usr/bin/sh
python /path/to/acme_tiny_dns.py --account-key /path/to/account.key --csr /path/to/domain.csr --domain "${domain}" --hook /path/to/hook/script > /path/to/signed_chain.crt || exit
service nginx reload
```

```
# Example line in your crontab (runs once per month)
0 0 1 * * /path/to/renew_cert.sh 2>> /var/log/acme_tiny.log
```

## Permissions

If you can, try to run acme_tiny_dns.py as a non privileged user.

You can for example run this as root:

```
acme_bin="/usr/local/bin/acme_tiny_dns.py"
hook_bin="/usr/local/bin/update_my_dns"
letsencrypt_account_key="/etc/letsencrypt/user.key"
letsencrypt_user="letsencrypt"
domain="example.org"
domain_csr="/etc/letsencrypt/${domain}.csr"
domain_crt="/etc/nginx/ssl/${domain}.crt"

su -s /bin/sh -c "umask 033; python ${acme_bin} --account-key \"${letsencrypt_account_key}\" --csr \"${domain_csr}\" --domain \"${domain}\" --hook \"${hook_bin}\" > \"${domain_crt}\"" "${letsencrypt_user}"
```

The biggest problem you'll likely come across while setting up and running this
script is permissions. You want to limit access to your account private key and
challenge web folder as much as possible. I'd recommend creating a user
specifically for handling this script, the account private key, and the
challenge folder. Then add the ability for that user to write to your installed
certificate file (e.g. `/path/to/signed_chain.crt`) and reload your webserver. That
way, the cron script will do its thing, overwrite your old certificate, and
reload your webserver without having permission to do anything else.

**BE SURE TO:**
* Backup your account private key (e.g. `account.key`)
* Don't allow acme_tiny_dns.py script to be able to read your domain private key!
* Try not to run acme_tiny_dns.py as root!
