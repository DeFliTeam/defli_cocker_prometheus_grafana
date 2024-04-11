# defli_docker_prometheus_grafana 

For windows users: 

### Install WSL with command 

```bash 

install --wsl 

``` 

### Open windows terminal as as administrator and run command, also open an ubuntu terminal 

```bash 
usbipd list
``` 

### Identify your SDR, it is normally referred to as "bulk interface" note the BUS ID, it typically takes the format of "1-1", "1-2", "1-x" 

### Run command replacing the "4-4" with your device BUS ID 

```bash
usbipd bind --busid 4-4
```

### Close the windows terminal and open a new one without administrator function 

### Run command below, again replacing "4-4" with your own BUS ID

```bash 
usbipd attach --wsl --busid 4-4
```
### Note that using WSL will assign you an IP address and this must be used in place of "localhost" in any of the following configuration.

### From Ubuntu Terminal (Note Ubuntu Users can start from here) 

### Run command



```bash
bash <(wget -q -O - https://raw.githubusercontent.com/sdr-enthusiasts/docker-install/main/docker-install.sh)
``` 

### then 

```bash 
sudo reboot
``` 

### Next 

```bash 
sudo mkdir -p -m 777 /opt/adsb
``` 

```bash 
cd /opt/adsb
``` 

```bash 
sudo wget https://raw.githubusercontent.com/sdr-enthusiasts/docker-adsb-ultrafeeder/main/docker-compose.yml
``` 

```bash
sudo wget https://raw.githubusercontent.com/sdr-enthusiasts/docker-adsb-ultrafeeder/main/.env
``` 

### We are now going to replace the contents of the docker-compose.yml file and the .env file 

```bash 
sudo nano docker-compose.yml
``` 

### Once opened delete the contents and paste in the below. Add in your Lat, Lon, elevation (in M) and TZ where required using this format 

- TZ=your_input
- READSB_LAT=your_latitude 
- READSB_LON=your_longitude
- READSB_ALT=110
- TAR1090_DEFAULTCENTERLAT=your_latitude
- TAR1090_DEFAULTCENTERLON=your_longitude

```bash
services:
  ultrafeeder:
    image: ghcr.io/sdr-enthusiasts/docker-adsb-ultrafeeder:telegraf
    tty: true
    container_name: ultrafeeder
    hostname: ultrafeeder
    restart: unless-stopped
    device_cgroup_rules:
      - "c 189:* rwm"
    ports:
      - 8080:80 # to expose the web interface
      - 9273-9274:9273-9274 # to expose the statistics interface to Prometheus
    environment:
      # --------------------------------------------------
      # general parameters:
      - LOGLEVEL=error
      - TZ=YOUR TIMEZONE # enter your timezone using https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
      # --------------------------------------------------
      # SDR related parameters:
      - READSB_DEVICE_TYPE=rtlsdr
      #
      # --------------------------------------------------
      # readsb/decoder parameters:
      - READSB_LAT=FEEDER_LAT # enter your latitude
      - READSB_LON=FEEDER_LONG # enter your longitude
      - READSB_ALT=FEEDER_ALT_Mm # enter your elevation
      - READSB_GAIN=autogain
      - READSB_RX_LOCATION_ACCURACY=2
      - READSB_STATS_RANGE=true
      #
      # --------------------------------------------------
      # TAR1090 (Map Web Page) parameters:
      - UPDATE_TAR1090=true
      - TAR1090_DEFAULTCENTERLAT=FEEDER_LAT # enter your latitude
      - TAR1090_DEFAULTCENTERLON=FEEDER_LONG # enter your longitude
      - TAR1090_MESSAGERATEINTITLE=true
      - TAR1090_PAGETITLE=My_DeFli_Feeder
      - TAR1090_PLANECOUNTINTITLE=true
      - TAR1090_ENABLE_AC_DB=true
      - TAR1090_FLIGHTAWARELINKS=false
      - TAR1090_SITESHOW=true
      - TAR1090_RANGE_OUTLINE_COLORED_BY_ALTITUDE=true
      - TAR1090_RANGE_OUTLINE_WIDTH=2.0
      - TAR1090_RANGERINGSDISTANCES=50,100,150,200
      - TAR1090_RANGERINGSCOLORS='#1A237E','#0D47A1','#42A5F5','#64B5F6'
      - TAR1090_USEROUTEAPI=true
      #
      # --------------------------------------------------
      # GRAPHS1090 (Decoder and System Status Web Page) parameters:
      # The two 978 related parameters should only be included if you are running dump978 for UAT reception (USA only)
      - GRAPHS1090_DARKMODE=true
      #
      # --------------------------------------------------
      # Prometheus and InfluxDB connection parameters:
      - PROMETHEUS_ENABLE=true
      - PROMETHEUSPORT=9273
    volumes:
      - /opt/adsb/ultrafeeder/globe_history:/var/globe_history
      - /opt/adsb/ultrafeeder/graphs1090:/var/lib/collectd
      - /proc/diskstats:/proc/diskstats:ro
      - /dev:/dev:ro
    tmpfs:
      - /run:exec,size=256M
      - /tmp:size=128M
      - /var/log:size=32M
```

### Once you have done this, please use to save

```bash 
ctrl + x 
y 
``` 

### Now configure your .env file 

```bash 
sudo nano .env 
``` 

### Paste in the following using the values you entered in the above file 

```bash
FEEDER_TZ=
FEEDER_LAT=xx.xxxxxx
FEEDER_LONG=xx.xxxxxx
FEEDER_ALT_M=xx
FEEDER_NAME=my_defli_feeder
``` 

### Save the file with 

```bash
ctrl + x 
y 
```

### Start the container using 

```bash 
docker compose up -d ultrafeeder
``` 

### Back on the ubuntu command line run this command to create a grafana directory 

```bash 
cd .. 
cd .. 
```

```bash 
sudo mkdir -p -m777 /opt/grafana/grafana/appdata /opt/grafana/prometheus/config /opt/grafana/prometheus/data
cd /opt/grafana
sudo nano docker-compose.yml
``` 


### Paste in the following text 

```bash 
version: '3.9'

volumes:
  grafana:
    driver: local
    driver_opts:
      type: none
      device: "/opt/grafana/grafana/appdata"
      o: bind
  prom-config:
    driver: local
    driver_opts:
      type: none
      device: "/opt/grafana/prometheus/config"
      o: bind
  prom-data:
    driver: local
    driver_opts:
      type: none
      device: "/opt/grafana/prometheus/data"
      o: bind

services:
  grafana:
    image: grafana/grafana-oss:latest
    restart: unless-stopped
    container_name: grafana
    hostname: grafana
    tty: true
    # uncomment the following section and set the variables if you are exposing Grafana to the internet behind a rev web proxy:
    environment:
    # windrose panel plugin is needed for polar plots:
      - GF_INSTALL_PLUGINS=snuids-radar-panel,fatcloud-windrose-panel
    # uncomment and set the following variables if you are exposing Grafana to the internet behind a rev web proxy:
    #  - GF_SERVER_ROOT_URL=https://mywebsite.com/grafana/
    #  - GF_SERVER_SERVE_FROM_SUB_PATH=true
    # The following variables are needed if you want to expose and embed any dashboards publicly:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_NAME=public
      - GF_SECURITY_ALLOW_EMBEDDING=true
      - GF_PANELS_DISABLE_SANITIZE_HTML=true
      - GF_FEATURE_TOGGLES_ENABLE=publicDashboards
    # The following variables will allow you to "share/render" dashboards as PNG graphics.
    # You should also enabled the renderer container below.
      - GF_RENDERING_SERVER_URL=http://renderer:8081/render
      - GF_RENDERING_CALLBACK_URL=http://grafana:3000/
      - GF_LOG_FILTERS=rendering:debug
    ports:
      - 3000:3000
    volumes:
      - grafana:/var/lib/grafana

# The `renderer` container is needed if you want to share images of your dashboard as a graphic:
  renderer:
    image: grafana/grafana-image-renderer:latest

  prometheus:
    image: prom/prometheus
    container_name: prometheus
    hostname: prometheus
    restart: unless-stopped
    tmpfs:
      - /tmp
    volumes:
      - prom-config:/etc/prometheus
      - prom-data:/prometheus
    ports:
      - 9090:9090

```

### Start the container 

```bash
docker compose up -d
```

### We will now configure Prometheus 

 
### Prometheus needs to be told where to look for the data from the ultrafeeder. We will create a target prometheus configuration file that does this. 

### First we must determine the IP address of our ultrafeeder service 

```bash
docker exec -it ultrafeeder /bin/bash
```
```bash
cat /etc/hosts
```
This command should produce 7 lines of code. That last line should be an IP address starting "172". Please note this down. 

```bash
ctrl + d
```
### Now we create the config file

```bash 
docker exec -it prometheus sh -c "echo -e \"  - job_name: 'ultrafeeder'\n    static_configs:\n      - targets: ['172.17.0.1:9273', '172.17.0.1:9274']\" >> /etc/prometheus/prometheus.yml"
docker stop prometheus
docker compose up -d
```
Now add the remote write 

```bash
cd /opt/grafana/prometheus/config/
```
```bash
sudo nano prometheus.yml
```
Paste the below in to the bottom of the file
```bash
  remote_write:
  - url: https://prometheus-prod-13-prod-us-east-0.grafana.net/api/prom/push
    basic_auth:
      username: 1488847
      password: glc_eyJvIjoiMTA4MjgwNiIsIm4iOiJzdGFjay04ODc4MjAtaG0tcmVhZC1kZWZsaS1kb2NrZXIiLCJrIjoiN2NXNjJpMDkyTmpZUWljSDkwT3NOMDh1IiwibSI6eyJyIjoicHJvZC11cy1lYXN0LTAifX0=
```
```bash
ctrl + x
```
```bash
y
```
```bash
docker stop prometheus
```
```bash
docker compose up -d
```

### Now navigate to: 

http://you-ip-address:3000/   this is your personal grafana console username:admin password:admin

Click "add your first data source" 
Click "prometheus" 

Under name enter- ultrafeeder 
Under URL enter- http://prometheus:9090/ or http://your-ip-address:9090/ 

Click "save and test" 

If you get the green message you can click "build dashboard" 

In the box with title "import via grafana.com" enter 18398 and press load 

Select "ultrafeeder" from the dropdown list 

Click import 

Your dashboard will populate 

To add extra maps from tar1090 please follow these instructions https://github.com/DeFliTeam/defli_quickstart#incorporate-tar1090-and-graphs1090-in-to-grafana

### If you get a no data message on Grafana please take the following steps 

```bash
cd /opt/grafana/prometheus/config/
```
```bash
sudo nano prometheus.yml
```
Replace the 172 IP Address with your device IP address (likely starting 192)

```bash
ctrl + x
```
```bash
y
```
```bash
docker stop prometheus
```
```bash
docker compose up -d
```

### If you still get no data in Grafana please take the following steps 

```bash
cd /opt/grafana/prometheus/config/
```
```bash
sudo nano prometheus.yml
```
Replace the 172 or 192 IP Address with 172.17.0.1

```bash
ctrl + x
```
```bash
y
```
```bash
docker stop prometheus
```
```bash
docker compose up -d
```






### If all is working you should see outputs here: 
http://localhost:8080/ 
http://localhost:8080/graphs1090/
http://localhost:9273/metrics/ 
http://localhost:9090/

### Troubleshooting 

The best way to troubleshoot is to navigate to: 

http://localhost:9090/ or http://your-ip-address:9090/ 

Click on "Status" and then "targets" 

Look for the error codes: 

If you see "context deadline passed" then take the following steps 

```bash
cd /opt/grafana/prometheus/config/
```
```bash
sudo nano prometheus.yml
```
At the top of the file you will see "scrape_interval:  10s" change the "10" to "120"
```bash
ctrl + x
```
```bash
y
```
```bash
docker stop prometheus
```
```bash
docker compose up -d
```

If you get an error message saying "failed to connect" or "permission denied" then you are using the wrong IP address, switch to one of the other suggest 172 or 192 addresses in the prometheus file


If you get a "permission denied while tring to connect to the docker daemon socket error please use 

```bash
sudo groupadd docker
```
###Replace "username" with your own username this would be "defli" if your command line shows defli@DeFli

```bash
sudo usermod -aG docker username
```
```bash
newgrp docker
```

