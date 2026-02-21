# Cloudflare Setup

```
1. Blocks HTTP Connections
2. Blocks A huge amount of malicous of useragents that are either used for DDoSing or crawling.
3. Blocks anything that is above 1 threat level.
4. Blocks Spoofed IPS
5. Blocks any HTTP Version that is not 1.0, 1.1. 1.2, 2, or even 3
6. Blocks Tor
7. Blocks Unknown States
8. Blocks Known Bots
9. Blocks requests that are not GET, POST
10. Blocks request refers containing .xyz, .com, .edu, .gov, .biz, .io, .net, .pro, .info 
11. Blocks most public TLS methods used for DDoSing
```

```
(cf.client.bot) or (http.user_agent contains "Cyotek") or (http.user_agent contains "python") or (http.user_agent contains "undefined") or (http.user_agent eq "Empty user agent") or (http.user_agent contains "HTTrack") or (http.user_agent contains "Java") or (http.user_agent contains "curl") or (http.user_agent contains "RestSharp") or (http.user_agent contains "Ruby") or (http.user_agent contains "Nmap") or (http.user_agent eq "libwww") or (not http.request.version in {"HTTP/1.0" "HTTP/1.1" "HTTP/1.2" "HTTP/2" "HTTP/3"}) or (ip.geoip.country eq "T1") or (ip.geoip.country eq "XX") or (cf.threat_score ge 1) or (not http.request.method in {"GET" "POST"}) or (http.user_agent contains " Uptime-Kuma") or (http.user_agent contains "sitechecker") or (http.user_agent contains "axios") or (http.referer contains "fbi.com") or (http.user_agent eq " Stripe/1.0 (+https://stripe.com/docs/webhooks)") or (http.cookie eq "cf_rate=mjRCQSuBeJk3Db") or (http.user_agent contains "Python") or (http.user_agent contains "YaBrowser") or (http.user_agent contains "lt-GtkLauncher") or (http.user_agent contains "3gpp-gba") or (http.user_agent contains "UNTRUSTED") or (http.user_agent contains "Galeon/1.3.15") or (http.user_agent contains "Phoenix/0.2" and http.user_agent contains "Fennec/2.0.1") or (http.user_agent contains "Minefield") or (http.user_agent contains ".NET") or (http.user_agent contains " Arora/0.8.0") or (http.referer eq "https://check-host.net") or (http.user_agent contains "https://check-host.net") or (http.user_agent contains "CheckHost") or (ip.src in {0.0.0.0/8 10.0.0.0/8 100.64.0.0/10 127.0.0.0/8 127.0.53.53 169.254.0.0/16 172.16.0.0/12 192.0.0.0/24 192.0.2.0/24 192.168.0.0/16 198.18.0.0/15 198.51.100.0/24 203.0.113.0/24 224.0.0.0/4 240.0.0.0/4 255.255.255.255/32}) or (http.user_agent eq "-") or (http.user_agent eq " ") or (http.user_agent eq "nginx-ssl early hints")
```

# ASN Blocking
```
(ip.geoip.asnum in {16276})
```
https://github.com/NullifiedCode/ASN-Lists