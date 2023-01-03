## About This NGINX Configuration
The hardening for this NGINX server was modeled based on:
https://github.com/AdrienKuhn/hardened-nginx

NGINX-extras implementation from:
https://github.com/byjg/docker-nginx-extras

The init script for generating a temporary certificate (used on a fresh build) is located at /system/letsencrypt/init-letsencrypt.sh and was borrowed from:
https://github.com/wmnnd/nginx-certbot

Dry runs for the script require the ./system/certbot/conf folder to exist. To regenerate the certificate delete the conf folder, and recreate it.

Always test with staging set to 1. If it succeeds, then change back to 0.

The Dockerfile for this server extends the base image and copys these files over:
/system/nginx/jarvis.conf
/system/nginx/http.conf
/system/nginx/nginx.conf
/system/nginx/ssl.conf
/system/nginx/stream.conf