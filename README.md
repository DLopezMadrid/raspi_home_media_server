# Pi 4 home media server  


## Hardware  
- Raspberry pi4 (4GB)
- 1TB external HDD (WD blue) 

## Software  
- [Dietpi v6.32.2](https://dietpi.com/)
- [Docker](https://www.docker.com/)
- [Docker-compose](https://docs.docker.com/compose/install/)
- [Ddclient](https://github.com/ddclient/ddclient)
- [Pihole](https://pi-hole.net/)  

## Docker images (with links)
- [Sonarr](https://hub.docker.com/r/linuxserver/sonarr)
- [Radarr](https://hub.docker.com/r/linuxserver/radarr)
- [Jackett](https://hub.docker.com/r/linuxserver/jackett)
- [Deluge](https://hub.docker.com/r/linuxserver/deluge)
- [Plex](https://hub.docker.com/r/linuxserver/plex)
- [Organizr](https://hub.docker.com/r/linuxserver/organizr)  
- [Tautulli](https://hub.docker.com/r/linuxserver/tautulli)  
- [Nginx+letsencrypt (swag)](https://hub.docker.com/r/linuxserver/swag)  
- [NordVPN client](https://hub.docker.com/r/bubuntux/nordvpn)    

## Other
- Namecheap domain
- Nordvpn account
- Home assistant (hassio) running in another raspi 3b

## Resources
[https://github.com/thundermagic/rpi\_media\_centre](https://github.com/thundermagic/rpi_media_centre)  
[https://github.com/marchah/pi-htpc-download-box](https://github.com/marchah/pi-htpc-download-box)  
[https://github.com/sebgl/htpc-download-box](https://github.com/sebgl/htpc-download-box)  
[https://github.com/nagyben/oracle](https://github.com/nagyben/oracle)
[https://daan.dev/how-to/reverse-proxy-omv-letsencrypt-sabnzbd-radarr-sonarr-transmission/3/](https://daan.dev/how-to/reverse-proxy-omv-letsencrypt-sabnzbd-radarr-sonarr-transmission/3/)


## Installation
Most of this software is dockerized and launched using docker-compose. The only things that run in the pi natively are dietpi (obviously), pihole (from dietpi-software and out-of-scope for this doc) and ddclient.

The external HDD is mounted into /mnt/media and has the following directory structure

```bash
/mnt/media  
    ├── appdata
    ├── downloads
    ├── movies
    ├── music
    ├── pictures
    └── tv_shows
```

Within appdata we create individual folders to store the config files of each app
```bash
/mnt/media/appdata
    ├── jackett
    ├── lidarr
    ├── plex
    ├── radarr
    ├── sonarr
    ├── tautulli
    ├── nginx
    └── etc...
```
Most of the containers that I use are from [linuxserver.io](https://www.linuxserver.io/). The `docker-compose.yaml` file contains all the configs for them. The only one not from linuxserver.io is the NordVPN client one (from [bubuntux](https://hub.docker.com/r/bubuntux/nordvpn)), the reason for this is because this NordVPN client supports their new NordLynx (wireguard) protocol, which is supposed to be faster.


## Installation


#### Preparation
Clone this repo and create the directory structure described in the previous section  


#### Install docker:
```bash
$ curl -fsSL https://get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh
$ sudo groupadd docker
$ sudo usermod -aG docker $USER
```
These steps are from: https://docs.docker.com/install/linux/docker-ce/debian/#install-using-the-convenience-script

#### Install python 3 and python 3 pip
```bash
$ sudo apt install python3 && sudo apt install python3-pip
```

#### Install docker-compose
```bash
$ python3 -m pip install docker-compose
```

#### Check that docker and docker-compose are working
Reboot and run these commands, if you the regular expected output, everything is installed properly
```bash
$ docker version
$ docker-compose version
```

#### Launch docker-compose
Go to to the repo folder and check that the `.env` and `docker-compose.yaml` files match your config, follow the instruction on the files to find your specific values.

Once this is ready we can launch docker-compose using

```bash
$ docker-compose up -d
```
This will start pulling all the docker images and configure them using the values on the `docker-compose.yaml` file

Everything should be up 'n' running once the command has finished executing

##### Useful commands

See all containers running
```bash
$ docker container ls
```

You can check individual logs using 
```bash
$ docker container logs <container_name>
```

Restart a container
```bash
$ docker-compose restart -t 1 <container_name>
```

## Configuration

Follow [these steps](https://github.com/sebgl/htpc-download-box) to configure each container


## Reverse proxy
This is just for extra points, you should comment out the swag container out of `docker-compose.yaml` if you are not using a custom domain to host your apps


#### Install & run ddclient (dynamic IP)
We will use this to update our dynamic IP on our domain provider (namecheap in my case)
```bash
$ sudo apt install ddclient
```
Ignore the setup wizard and choose any values.  

ddclient will read the config from `/etc/ddclient.conf`, copy the repo's `ddclient.conf` content into it 

Follow [this tutorial](https://www.namecheap.com/support/knowledgebase/article.aspx/583/11/how-do-i-configure-ddclient) from namecheap to fill the config file with your secret. Remember to include `@, *` in the last line of your config file

In the namecheap website, go to the Advanced DNS section, enable dynamic IP and create A + Dynamic DNS Records for `@, *` and set any IP address (the IP will get updated once you run ddclient)(I had to create individual ones for each subdomain that I wanted to use but in principle it should not be necessary, if you do it just make sure that the subdomains are also included in the last line of the `ddclient.conf` file)

Run ddclient
```bash
$ sudo ddclient -v
```
You should see a success message in the terminal and the updated IP address in the A + Dynamic DNS Records in the namecheap web

If that worked, make sure you add ddclient to the sudo crontab

```bash
$ sudo crontab -e
```
Add this to run ddclient every 7 mins and after every reboot
```bash
*/7 * * * * sudo ddclient
@reboot sudo ddclient
```
#### Nginx reverse proxy
- Configure the apps to use SSL using [this guide](https://daan.dev/how-to/reverse-proxy-omv-letsencrypt-sabnzbd-radarr-sonarr-transmission/3/)
- For Plex, you will need to disable remote access, add the custom URL to Settings/Network/Custom server access URL through the web and set Secure connections to Preferred.
- Reboot all the containers for the settings to be applied
- Open ports 80 & 443 in your router and assign them to your raspi.
- Copy the contents of the `nginx` folder in the repo (specially the `proxy-confs` folder) to the `swag` folder in `appdata`
- Inside `proxy-confs` there will be a .conf file for each app, make sure that the IPs are correct for your setup (192.168.1.100 in my case) 
- Restart the `swag` container and check the logs, you should see it acquiring the certificates correctly and once that is done you should be able to access the apps from the external URLs
- Don't forget to set up passwords for whatever apps are exposed to the outside
- You can also use nginx to assign custom domains to other raspis in your home network (e.g. Hassio)

## Android app
Use [NZB360](https://play.google.com/store/apps/details?id=com.kevinforeman.nzb360&hl=en_GB)