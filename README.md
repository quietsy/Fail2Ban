# Securing SWAG

SWAG - Secure Web Application Gateway (formerly known as letsencrypt) is a full fledged web server and reverse proxy with Nginx, Php7, Certbot (Let's Encrypt™ client) and Fail2ban built in.

SWAG allows you to expose applications to the internet, doing so comes with a risk and there are security measures that help reduce that risk. This article details how to configure SWAG and enhance it's security.

This article assumes that you already have a functional SWAG setup. Following is the compose yaml used to create the SWAG container referenced in this article. Keep in mind your local mount paths will be different so adjust accordingly.

```YAML
---
version: "2.1"
services:
  swag:
    image: linuxserver/swag
    container_name: swag
    cap_add:
      - NET_ADMIN
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
      - URL=linuxserver-test.com
      - SUBDOMAINS=wildcard
      - VALIDATION=dns
      - DNSPLUGIN=cloudflare #optional
      - PROPAGATION= #optional
      - DUCKDNSTOKEN= #optional
      - EMAIL= #optional
      - ONLY_SUBDOMAINS=false #optional
      - EXTRA_DOMAINS= #optional
      - STAGING=false #optional
    volumes:
      - /home/user/swag:/config
    ports:
      - 443:443
      - 80:80 #optional
    restart: unless-stopped
```

### Fail2Ban

Fail2Ban is an intrusion prevention software that protects external applications from brute-force attacks. Attackers who fail to login to your applications a certain number of times will get blocked from accessing all of your applications.

For example, in order to protect Nextcloud create a file called nextcloud.conf under fail2ban/filter.d:

```
[Definition]
failregex=^.*Login failed: '?.*'? \(Remote IP: '?<ADDR>'?\).*$
          ^.*\"remoteAddr\":\"<ADDR>\".*Trusted domain error.*$
ignoreregex =
```

Add the following configuration to fail2ban/jail.local:

```
[nextcloud]
enabled = true
port     = http,https
filter = nextcloud
logpath = /nextcloud/nextcloud.log
action  = iptables-allports[name=nextcloud]
```

Finally add the following volume to the compose yaml:
```
      - /path/to/nextcloud/nextcloud.log:/nextcloud/nextcloud.log:ro
```

Repeat the process for every external application, you can find Fail2Ban configurations for most applications on the internet.

This great mod sends a discord notification when Fail2Ban blocked an attack: [f2bdiscord](https://github.com/linuxserver/docker-mods/tree/swag-f2bdiscord)

### Geoblock

Geoblock is a great way to reduce the attack surface of SWAG by restricting access based on countries.

To enable geoblock, uncomment the Geoip2 config line in nginx.conf:
```
include /config/nginx/geoip2.conf;
```

Acquire a Maxmind license key [here](https://www.maxmind.com/en/geolite2/signup)

Add the -e MAXMINDDB_LICENSE_KEY=<licensekey> to the compose yaml to automatically download the Geolite2 database.

Below are 2 examples:
- Allowing a single country and your LAN.
- Allowing everything except high risk countries. (GilbN's list based on the Spamhaus statistics and Aakamai’s state of the internet report)

```Nginx
map $geoip2_data_country_iso_code $allowed_mycountry {
    default no;
    US yes; #Replace with your country code list https://dev.maxmind.com/geoip/legacy/codes/iso3166/
    192.168.1.0/24 yes; #Replace with your LAN subnet
}

map $geoip2_data_country_iso_code $denied_highrisk {
    default yes; #If your country is listed below, remove it from the list
    CN no; #China
    RU no; #Russia
    HK no; #Hong Kong
    IN no; #India
    IR no; #Iran
    VN no; #Vietnam
    TR no; #Turkey
    EG no; #Egypt
    MX no; #Mexico
    JP no; #Japan
    KR no; #South Korea
    KP no; #North Korea
    PE no; #Peru
    BR no; #Brazil
    UA no; #Ukraine
    ID no; #Indonesia
    TH no; #Thailand
 }
```

The final step is to utilize the geoblock in your configuration, add one of the following lines above your location section in every application you want to protect.
```
    if ($allowed_mycountry = no) { return 404; }
    if ($denied_highrisk = no) { return 404; }
```

Example:

```Nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name authelia.*;
    include /config/nginx/ssl.conf;
    client_max_body_size 0;

    if ($allowed_mycountry = no) { return 404; }

    location / {
        include /config/nginx/proxy.conf;
        resolver 127.0.0.11 valid=30s;
        set $upstream_app authelia;
        set $upstream_port 9091;
        set $upstream_proto http;
        proxy_pass $upstream_proto://$upstream_app:$upstream_port;
    }
}
```

Add the line to every external application based on your needs.

### NGINX Configuration

##### HSTS
HTTP Strict Transport Security (HSTS) is a web security policy mechanism that helps to protect websites against man-in-the-middle attacks such as protocol downgrade attacks and cookie hijacking. It allows web servers to declare that web browsers (or other complying user agents) should automatically interact with it using only HTTPS connections, which provide Transport Layer Security (TLS/SSL), unlike the insecure HTTP used alone.

To enable, uncomment the HSTS config line in ssl.conf:
```
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```

##### X-Robots-Tag
You can prevent applications from appearing in results of search engines and web crawlers, regardless of whether other sites link to it. It doesn't work on all search engines and web crawlers, but it significantly reduces the chance.

To enable, uncomment the X-Robots-Tag config line in ssl.conf:
```
add_header X-Robots-Tag "noindex, nofollow, nosnippet, noarchive";
```

To disable on a specific application and allow search engines to display it, add the following line to the application config inside the server tag:
```
add_header X-Robots-Tag "";
```

### Internal Applications

Some applications are only used internally in your network and don't have to be exposed to the internet, but you can still route them through SWAG and reach them with a local domain. This requires a local DNS, such as dnsmasq, unbound, pihole, adguardhome, etc.

Configure your local DNS to redirect *.local to SWAG, for example add the following line to dnsmasq.conf:
```
address=/local/192.168.1.5
```

Replace local with your desired domain (to avoid problems don't use existing domains like .com, .net, etc.), replace 192.168.1.5 with the IP of SWAG's host.

Make the following changes to the application's configuration:

```Nginx
server {
    listen 80; #local applications can listen on port 80
    server_name calibre.local; #explicitely state application.local
    client_max_body_size 0;

    location / {
        include /config/nginx/proxy.conf;
        resolver 127.0.0.11 valid=30s;
        set $upstream_app calibre;
        set $upstream_port 8080;
        set $upstream_proto http;
        proxy_pass $upstream_proto://$upstream_app:$upstream_port;
        proxy_buffering off;
    }
}
```

Repeat the process for all internal applications.

The recommended way to access internal applications from the internet is through a VPN, for example WireGuard:

[WireGuard Container](https://hub.docker.com/r/linuxserver/wireguard)

[WireGuard on OPNSense](https://blog.linuxserver.io/2019/11/16/setting-up-wireguard-on-opnsense-android/)

### Authelia

Authelia is an open-source authentication and authorization server providing 2-factor authentication and single sign-on (SSO) for your applications via a web portal. Refer to this  [blog post to configure Authelia](https://blog.linuxserver.io/2020/08/26/setting-up-authelia/).


