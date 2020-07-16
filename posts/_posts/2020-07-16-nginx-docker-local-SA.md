---
layout: post
title: Creating nginx SSL proxies for services in docker in LAN
---
This article talks about how one can use nginx to create SSL proxies for docker (and, potentially, other) services running in the local (e.g. home) network. Since the network is a home one, the certificates will be signed by specially created Certificate Authority that will be marked as trusted on the client devices.

<!--more-->
* Do not remove this line (it will not be displayed)
{:toc}

# Environment

In this scenario, we will have two machines. A **client** and a **server**. The **server** machine is running one or more Docker containers. These containers expose certain ports to manage or access the services running inside those containers.

For example, **server** could be running [Pi-hole](https://pi-hole.net/) and a [gogs](https://gogs.io/) instance. Pi-hole has a management console, available from client through <https://$SERVER_IP/admin> and gogs is available through <https://$SERVER_IP:3000>. In both cases, when client machine accesses these services through a browser, a warning page about the certificate is displayed.

In my environment, client machines are running Gentoo Linux and the server is Alpine Linux 3.11.6. Some configuration steps are distro-specific and may require some tweaking. The versions of software used:

* OpenSSL 1.1.1g  21 Apr 2020
* nginx version: nginx/1.16.1

# Overview

The steps are:

* Create a local [Certificate Authority](https://en.wikipedia.org/wiki/Certificate_authority)
* Add it to the list of trusted CAs on client machines
* Create certificates for the services that should have SSL proxies
* Set up nginx proxies for the services

# Creating local CA

Source article on WikiHow: [link](https://www.wikihow.com/Be-Your-Own-Certificate-Authority)

This procedure follows the source article, but a couple of additional steps are required due to default `openssl.conf` on Gentoo

On **client** or **server** machine execute the following commands as any user in any directory:

1. Create the key:

        openssl genrsa -des3 -out server.CA.key 2048
2. Create the certificate signing request:

        openssl req -verbose -new -key server.CA.key -out server.CA.csr -sha256

    Provide the information as it's being requested.

3. Create the CA certificate:

        openssl ca -extensions v3_ca -out server.CA-signed.crt -keyfile server.CA.key -verbose -selfsign -create_serial -rand_serial -md sha256 -enddate 330630235959Z -infiles server.CA.csr

This command is slightly different than the original article since it also uses `-create_serial` flag to automatically create the [serial number](https://www.ssl.com/faqs/what-is-an-x-509-certificate/#ftoc-heading-2).

If you get any errors at this stage such as:

    ca: ./demoCA/newcerts is not a directory
    ./demoCA/newcerts: No such file or directory

or

    140596453635904:error:02001002:system library:fopen:No such file or directory:crypto/bio/bss_file.c:69:fopen('./demoCA/index.txt','r')
    140596453635904:error:2006D080:BIO routines:BIO_new_file:no such file:crypto/bio/bss_file.c:76:

Create the necessary directory and fill in files. These requirements are due to the default configuration in `/etc/ssl/openssl.cnf`. To fix:

    mkdir demoCA/{newcerts,private}
    touch demoCA/index.txt

If you want to manually create the serial number, you can provide it in file `./demoCA/serial`. See **EXAMPLES** section of `man openssl-ca` for details.

One last step is required: the `.crt` file needs to be converted into `.pem`, as some commands later expect that format. To do that, run:

    openssl x509 -in server.CA-signed.crt -out server.CA-signed.pem -outform PEM

As a result of this step, a file `server.CA-signed.pem` will be created in your working directory.

## Verification

To inspect the created CA certificate, run

    openssl x509 -noout -text -in server.CA-signed.crt

The output should display information you provided on step 2.

# Adding new CA to the trusted list on a client

After the CA certificate is created, it is necessary to mark it as trusted in **client** OS and the browser. The exact steps are distro- and browser-specific, but quite a few distributions ship with `update-ca-certificates` utility that handles the OS level. Whether the CA certificate is marked as trusted on OS level will affect OS utilities that are using `openssl`, e.g. `curl`, `git`, etc. Browsers typically have their own CA store.

Move the `server.CA-signed.crt` file to the client machine into `/usr/local/share/ca-certificates` directory. As **root** run

    update-ca-certificates

It should report that a certificate has been added and should not display any warnings.

    Updating certificates in /etc/ssl/certs/...
    1 added, 0 removed; done
    Running hooks in /etc/ca-certificates/update.d...

Browser-specific steps vary greatly between browsers, but for Firefox, go to Preferences > Advanced > Certificates > View Certificates. In the Authorities tab, click on the Import button to open the dialog to import a certificate to the store.

More information on this step is available in [Gentoo Wiki](https://wiki.gentoo.org/wiki/Certificates).

# Creating certificates

The next step is to create the individual certificates for services running on **server**. In this case, they are Pi-hole and gogs. Pi-hole management console will be accessible under <https://pihole.home.local> and gogs will be as <https://git.home.local>. The use of `home.local` part is optional.

This step follows the answer on [StackOverflow](https://stackoverflow.com/questions/7580508/getting-chrome-to-accept-self-signed-localhost-certificate/43666288#43666288) and may be summarized by running the following script in your working folder with previously created CA files. I adjusted this script slightly to generate the certificates in their own folder.

The script:

```shell
#!/bin/sh

if [ "$#" -ne 1 ]
then
    echo "Generates certificate for provided domain using CA certificate in the same directory."
    echo "Usage: $(basename "$0") domain.tld"
    exit 1
fi

DOMAIN=$1

if [ ! -d "$DOMAIN" ]; then
    mkdir "$DOMAIN"
else
    echo "Domain directory already exists. You may want to remove it first"
    exit 1
fi

openssl genrsa -out "$DOMAIN/$DOMAIN".key 2048
openssl req -new -key "$DOMAIN/$DOMAIN".key -out "$DOMAIN/$DOMAIN".csr \
    -subj "/C=AU/ST=Some-State/L=/O=Internet Widgits Pty Ltd/CN=$DOMAIN"

cat > "$DOMAIN/$DOMAIN".ext << EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = $DOMAIN
EOF

openssl x509 -req -in "$DOMAIN/$DOMAIN".csr \
    -CA server.CA-signed.pem -CAkey server.CA.key -CAcreateserial \
    -days 8250 -sha256 -extfile "$DOMAIN/$DOMAIN".ext \
    -out "$DOMAIN/$DOMAIN".crt
```

The `-subj` usage on line 21 may be adjusted to suit your needs or to match the information provided in CA certificate.

Before running the script the directory looks as follows:

```
.
├── demoCA
│   ├── index.txt
│   ├── index.txt.attr
│   ├── index.txt.old
│   ├── newcerts
│   │   └── 3B8C4534EC103B6AA5B1A9DF7326CBFAD0B12D29.pem
│   ├── private
│   └── serial
├── generate.sh
├── server.CA.csr
├── server.CA.key
├── server.CA-signed.crt
├── server.CA-signed.pem
```

After running `./generate.sh git.home.local`:

```
.
├── demoCA
│   ├── index.txt
│   ├── index.txt.attr
│   ├── index.txt.old
│   ├── newcerts
│   │   └── 3B8C4534EC103B6AA5B1A9DF7326CBFAD0B12D29.pem
│   ├── private
│   └── serial
├── generate.sh
├── git.home.local
│   ├── git.home.local.crt
│   ├── git.home.local.csr
│   ├── git.home.local.ext
│   └── git.home.local.key
├── server.CA.csr
├── server.CA.key
├── server.CA-signed.crt
├── server.CA-signed.pem
└── server.CA-signed.srl
```

The script generated the private key(`git.home.local.key`) and the certificate (`git.home.local.crt`) in the corresponding folder.

## Verification

To verify that the CA file was correctly imported into OS store and the generated certificate is now trusted, run:

    openssl verify git.home.local/git.home.local.crt

If the verification failed, manually provide the CA file:

    openssl verify -CAfile server.CA-signed.pem git.home.local/git.home.local.crt

If this succeeds, double-check the steps in the previous section. If the verification fails again — something is wrong in this step. Try to follow the script line-by-line and check out StackOverflow link provided earlier.

# Setting up nginx

After nginx is installed on the **server**, as root edit the `/etc/nginx/nginx.conf` file and add the following lines after `ssl_prefer_server_ciphers on;`

```
        # Enables a shared SSL cache with size that can hold around 8000  sessions.
        ssl_session_cache shared:SSL:2m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        # Defining used protocol versions.
        ssl_ciphers HIGH:!aNULL:!eNULL:!EXPORT:!CAMELLIA:!DES:!MD5:!PSK:!RC4;
 ```

 Next, create directory for SSL certificate:

    mkdir /etc/nginx/ssl/

Copy the `git.home.local.crt` and `git.home.local.crt` files to `/etc/nginx/ssl` and make the `.key` file only user-readable

To create a configuration file for our proxy, create a `git.home.local.conf` file in `/etc/nginx/conf.d/`. The file name may be different, but has to end in `.conf`. Remove the default config from that location.

For Debian-based distributives, the config should be created in `/etc/nginx/sites-available`. To enable it, create a symlink in `/etc/nginx/sites-enabled`:

    ln -s /etc/nginx/sites-available/git.home.local.conf /etc/nginx/sites-enabled

The file should contain:

```
server {
	listen 80;
	server_name git.home.local;
	return 301 https://$host$request_uri;
}
server {
	listen 443 ssl;
	server_name git.home.local;
	ssl_certificate           /etc/nginx/ssl/git.home.local.crt;
	ssl_certificate_key       /etc/nginx/ssl/git.home.local.key;

	location / {
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;
		proxy_pass http://git.home.local:3000;
		proxy_read_timeout 90;
	}
}
```

Repeat the steps in this section for other services.

This config sets up nginx to listen on ports 80 and 443. If some client accesses the sever via port 80, the client will be redirected to port 443. The `location /` section defines the proxy configuration. `proxy_pass` value may depend on your existing DNS setup and the port value defined in Docker configuration.

If using docker-compose to run Docker containers, the outside port is the first one in the string. In my `pihole.home.local` configuration:

```
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "67:67/udp"
      - "8080:80/tcp"
      - "8443:443/tcp"
```

So the `proxy_pass` is set to

		proxy_pass http://pihole-major.home.local:8080;

In my case, local DNS resolves both `pihole.home.local` and `git.home.local` to the `$SERVER_IP` and the firewall on the server allows communications from local area network, 192.168.1.0/24. If using a separate DNS server for your local network, make sure `nginx` has proper value set in `resolvers` or use the IP address instead of the hostname in `proxy_pass`.

The connection sequence goes as follows:

    Client <-> Server:80 (nginx) -> Server:443 (nginx) <-> Server:3000 (gogs)

Call `nginx -t` to verify and apply config.

## Verification

To test individual stages of this configuration, try accessing the defined hostname from client machine.

1. `curl http://git.home.local` should return 301 redirect. If it does not, there is a problem in nginx configuration. Probably in the `server{listen 80;}` section or the global one.
2. `curl https://git.home.local` should return the content from the service running in Docker.

    If there is an SSL error: run `curl -k -vvv https://git.home.local` and examine the handshake. Probably a wrong certificate was supplied, or CA was not added to the OS store.
    If the connection hangs: probably the problem is the value of `proxy_pass`. Try running on server: `telnet git.home.local 3000` to check connectivity between the proxy and the Docker service.

3. Try accessing <https://git.home.local> in browser running on the **client**. If there are any errors only on this stage — probably CA is not trusted on the browser level.
