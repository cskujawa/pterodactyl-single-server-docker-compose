<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
      <ul>
        <li><a href="#built-with">Built With</a></li>
        <li><a href="#disclaimerh">Disclaimer</a></li>
        <li><a href="#file-descriptions">File Descriptions</a></li>
        <li><a href="#hurdles-&-notes">Hurdles & Notes</a></li>
      </ul>
    </li>
    <li>
      <a href="#contributing">Contributing</a>
    </li>
  </ol>
</details>

<!-- ABOUT THE PROJECT -->
## About The Project

This project is intended to be a reference for setting up Pterodactyl game server manager on a single server. That means running Pterodactyl Panel and Pterodactyl Wings on the same server. I did this all behind an NGINX reverse proxy and using SSL.

More information on the Pterodactyl Panel and Wings can be found here:
https://pterodactyl.io/

Getting this working did take quite a bit of time, and I wanted to share some files and notes as references in the hopes that it could save some people time in the future.

Big shout out to EdyTheCow for his examples of Dockerized Pterodactyl, they were instrumental in helping me get setup.

https://github.com/EdyTheCow/docker-pterodactyl

While I cannot guarantee that the files or notes in here will work for every environment, hopefully they will aid in understanding.

If you're here and reading this, let me know if there's more information we can provide to people on similar adventures.

<p align="right">(<a href="#top">back to top</a>)</p>


### Built With

The Pterodactyl Panel

https://github.com/pterodactyl/panel

Pterodactyl Wings

https://github.com/pterodactyl/wings

MySQL DB used to store Panel data

Redis is used as a cache by the Panel

NGINX is what allows web access to the containers and local communication between containers.

* [![MySQL][Mysql.com]][Mysql-url]
* [![Redis][Redis.io]][Redis-url]
* [![NGINX][NGINX.com]][Nginx-url]

<p align="right">(<a href="#top">back to top</a>)</p>


### Disclaimer

While I would love to setup a fully functional standalone docker-compose environment that can be deployed with a single command, I embarked on this Pterodactyl adventure to play games with my friends, and that's what I would like to do after having spent 3 or so weeks getting this operational.

I will likely be back here to refine and test this setup to do just that.

For now these files are trimmed down versions of what exist in a menagerie of projects I call my server. There may be extranneous commands/variables/configurations that are not required but still exist.

<p align="right">(<a href="#top">back to top</a>)</p>


### File Descriptions

- The docker-compose file
  - This is the engine, the heart and soul of my server. Without it, my server is simply a Ubuntu 22.04 server. This file contains the instructions for creating the containers for MySQL, Redis, NGINX, Certbot, Pterodactyl Panel, and the Pterodactyl Wings.
  - The commands in the MySQL section are intended to optimize the default MySQL containers memory usage which is attrocious by default.
  - The NGINX section may seem simple, but I advise you check the readme in the system/nginx directory for more information on how that functions.
- .env.example
  - This file contains the required information for the MySQL environment
- ./game-servers/panel/var/.env.example
  - This is a pretty standard .env file for Laravel. There is a few important things to note in here though. The LE_EMAIL, TRUSTED_PROXIES, and SESSION_SECURE_COOKIE settings gave me some headaches, those are what I eventually landed on.
- ./game-servers/wings/data/wings/etc/config.yml
  - This file is a little weird, when you get the Panel up and running and create the location then the node, it generates a config for you which you're intended to paste into this file, when you first start panel (if everything works) it will fill in the rest of the file for you. That being said there was a few thigns I had to manually configure to get everything working.
  - First is in the api section, the range used in this is the one assigned in the docker-compose file for the docker container network where panel/wings exist. 
    - trusted_proxies:
    - \- 172.30.0.1/16
  - At the bottom there is these two settings
    - allowed_origins:
    - \- https://panel.example.com
    - \- https://172.30.0.1/16
    - allow_cors_private_network: true
- ./system/nginx/pterry.conf
  - This file is by and far where I spent the most time working on this, and I'll go into greater detail on some aspects but for now I'll cover the high level topics.
  - This file has a lot of security measures in it and handling for SSL with real certificates.
  - Probably the most important thing to note is that while the wings container is running on port 8083 that port is not exposed by the wings container, but rather by NGINX. So NGINX had to be configured to listen on this port, and the reason being is that the Pterodactyl panel always writes :8083 into the URL when sending data to wings.

<p align="right">(<a href="#top">back to top</a>)</p>


## Hurdles & Notes

This section is just a random hodge-podge list of things I wish were in front of me when starting.

- MySQL setup, since the docker containers IP changes, I believe @'%' is necessary for the pterodactyl user.
  ```
  CREATE USER 'pterodactyl'@'%' IDENTIFIED BY 'passwordgoeshere';
  CREATE DATABASE panel;
  GRANT ALL PRIVILEGES ON panel.* TO 'pterodactyl'@'%' WITH GRANT OPTION;
  FLUSH PRIVILEGES;
  ```
- If you're using the example .env file provided you may need to delete the line for APP_KEY, then run this, take the key, and then add APP_KEY= with the key you copied from the command.
  ```
  docker-compose exec panel php artisan key:generate --show
  ```
- If you make any changes to the .env for panel ensure you clear the cache.
  ```
  docker-compose exec panel php artisan config:cache
  ```
- If you aren't 100% confident that the config you provided in the .env file, follow the instructions found [here](https://pterodactyl.io/panel/1.0/getting_started.html) and run these commands
  ```
  docker-compose exec panel php artisan p:environment:setup
  docker-compose exec panel php artisan p:environment:database
  docker-compose exec panel php artisan p:environment:mail
  ```
- If you create a server or node and then can't delete it for some reason, or you for any reason need to to wipe the database you can run
  ```
  docker-compose exec panel php artisan migrate:fresh
  ```
- If you're on a brand new DB or you've wiped the DB you'll likely need to seed the DB with the default data Pterodactyl provides
  ```
  docker-compose exec panel php artisan migrate --seed --force
  ```
- I'm not entirely certain this is needed, but at some point I ran it, so dropping here for the record.
  ```
  docker-compose exec panel composer install --no-dev --optimize-autoloader
  ```
- Although it's not documented incredibly well, all of the requests that the Panel makes to wings are across the port you assign to the node when creating it. For me this meant that I didn't need the listen 443 in NGINX for the wings server but rather just a listen 8083 ssl.
- Live communication between the Panel and Wings uses wss:// and that means that the requests routed through NGINX need to include these lines (specific for websocket routing.) The variable $http_upgrade is assigned at the top of the NGINX pterry.conf file.
  ```
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "Upgrade";
  ```
- The [webserver configuration](https://pterodactyl.io/panel/1.0/webserver_configuration.html) doc includes adding headers for NGINX routing, which should definitely be included.
- In my environment I didn't actually download or install the Panel or Wings, so using fastcgi_pass or anything fastcgi wouldn't work for me as (I understand it) the files would need to exist inside my server and be shared with the NGINX container which they are not.
- NGINX won't want to start if the containers it's trying to route to aren't up, so during initial startup it may restart a few times until they respond.
- In the docker-compose file, this folder needs to exist locally, for me I just created an empty directory `/var/lib/pterodactyl/:/var/lib/pterodactyl/` 
- When rebuilding a container (wings in this example) use these versions of the commands to ensure a clean rebuild
  ```
  docker-compose stop wings
  docker-compose build wings --no-cache
  docker-compose up -d wings --force-recreate
  ```
- When you create the node and have to assign allocations, I believe you need to use the servers internal IP address. That is what I ended up using. When attempted with my public IP I was getting 
  ```
  Failed to allocate and map port 7777-7777: Error starting userland proxy: listen tcp4 12.34.56.78:7777: bind: cannot assign requested address"
  ```
  When using 127.0.0.1 the server started and looked fully functional through the panel but was inaccessible from anywhere.
- I have my A records for the panel and wings subdomains routed to the same server.
- If you use UFW, ensure the docker containers can communicate with each other and the wings containers, and make sure the ports game servers need to access are open as well.


<p align="right">(<a href="#top">back to top</a>)</p>

<!-- CONTRIBUTING -->
## Contributing

Contributions are what make the open source community such an amazing place to learn, inspire, and create. Any contributions you make are **greatly appreciated**.

If you have a suggestion that would make this better, please fork the repo and create a pull request. You can also simply open an issue with the tag "enhancement".
Don't forget to give the project a star! Thanks again!

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

<p align="right">(<a href="#top">back to top</a>)</p>

<!-- MARKDOWN LINKS & IMAGES -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->

[Laravel.com]: https://img.shields.io/badge/Laravel-FF2D20?style=for-the-badge&logo=laravel&logoColor=white
[Laravel-url]: https://laravel.com
[Mysql.com]: https://img.shields.io/badge/MySQL-005C84?style=for-the-badge&logo=mysql&logoColor=white
[Mysql-url]: https://mysql.com
[Redis.io]: https://img.shields.io/badge/redis-%23DD0031.svg?style=for-the-badge&logo=redis&logoColor=white
[Redis-url]: https://Redis.io
[NGINX.com]: https://img.shields.io/badge/nginx-%23009639.svg?style=for-the-badge&logo=nginx&logoColor=white
[Nginx-url]: https://nginx.com
