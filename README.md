### Note
This was forked from DyonR's [docker-Jackettvpn](https://github.com/DyonR/docker-Jackettvpn) repo and includes the following changes from gjeanmart's [docker-Jackettvpn](https://github.com/gjeanmart/docker-Jackettvpn) repo:

`openvpn/start.sh` now contains the following to avoid the `OpenVPN - ERROR: Cannot open TUN/TAP dev /dev/net/tun: No such file or directory (errno=2)` error:

```
	# If create_tun_device is set, create /dev/net/tun
	if [[ "${CREATE_TUN_DEVICE,,}" == "true" ]] && [[ ! -f /dev/net/tun ]]; then
	  mkdir -p /dev/net
	  mknod /dev/net/tun c 10 200
	  chmod 0666 /dev/net/tun
	fi
```

`jackett/iptables.sh` / `Dockerfile` are based on an older commits, [iptables.sh](https://github.com/DyonR/docker-Jackettvpn/blame/215cf39a866b0f68af3e2798397e794ad729b0fe/jackett/iptables.sh) and [Dockerfile](https://github.com/gjeanmart/docker-Jackettvpn/commit/cd71c0a4d80a7c90a3a3143434859516953e5cfa), but avoids the `write UDP: Operation not permitted (code=1)` error. Once I bumped this to using ARM64, I opted to use the image that DyonR is currently using, `debian:bullseye-slim`, as it's actively maintained and half as large.

...and finally, instead of the AMDx64 build of Jackett, this uses ARM64.

# [Jackett](https://github.com/Jackett/Jackett) and OpenVPN
[![Docker Pulls](https://img.shields.io/docker/pulls/dyonr/jackettvpn)](https://hub.docker.com/r/dyonr/jackettvpn)
[![Docker Image Size (tag)](https://img.shields.io/docker/image-size/dyonr/jackettvpn/latest)](https://hub.docker.com/r/dyonr/jackettvpn)

Docker container which runs the latest headless [Jackett](https://github.com/Jackett/Jackett) Server while connecting to OpenVPN with iptables killswitch to prevent IP leakage when the tunnel goes down.

[preview]: https://raw.githubusercontent.com/DyonR/docker-templates/master/Screenshots/jackettvpn/jackettvpn-mainpage.png "Jackett Preview"
![alt text][preview]

## Docker Features
* Base: Debian bullseye-slim
* Latest [Jackett](https://github.com/Jackett/Jackett)
* Selectively enable or disable OpenVPN support
* IP tables kill switch to prevent IP leaking when VPN connection fails
* Configurable UID and GID for config files and /blackhole for Jackett
* Created with [Unraid](https://unraid.net/) in mind

# Run container from Docker registry
The container is available from the Docker Hub, which is the simplest way to get it.
To run the container use this command, with additional parameters, please refer to the Variables, Volumes, and Ports section:

```
$ docker run --privileged  -d \
              -v /your/config/path/:/config \
              -v /your/downloads/path/:/blackhole \
              -e "VPN_ENABLED=yes" \
              -e "VPN_TYPE=openvpn" \
              -e "LAN_NETWORK=192.168.0.0/24" \
              -p 9117:9117 \
              --restart unless-stopped \
              dyonr/jackettvpn
```

# Variables, Volumes, and Ports
## Environment Variables
| Variable | Required | Function | Example | Default |
|----------|----------|----------|----------|----------|
|`VPN_ENABLED`| Yes | Enable VPN? (yes/no)|`VPN_ENABLED=yes`|`yes`|
|`VPN_TYPE`| Yes | OpenVPN (openvpn)|`VPN_TYPE=openvpn`|
|`VPN_USERNAME`| No | If username and password provided, configures ovpn file automatically |`VPN_USERNAME=ad8f64c02a2de`||
|`VPN_PASSWORD`| No | If username and password provided, configures ovpn file automatically |`VPN_PASSWORD=ac98df79ed7fb`||
|`WEBUI_PASSWORD`| Yes | The password used to protect/access Jackett's web interface |`WEBUI_PASSWORD=RJayoLnKPjeyHbo-_ziH`||
|`LAN_NETWORK`| Yes (atleast one) | Comma delimited local Network's with CIDR notation |`LAN_NETWORK=192.168.0.0/24,10.10.0.0/24`||
|`NAME_SERVERS`| No | Comma delimited name servers |`NAME_SERVERS=1.1.1.1,1.0.0.1`|`1.1.1.1,1.0.0.1`|
|`PUID`| No | UID applied to config files and blackhole |`PUID=99`|`99`|
|`PGID`| No | GID applied to config files and blackhole |`PGID=100`|`100`|
|`UMASK`| No | |`UMASK=002`|`002`|
|`WEBUI_PORT`| No | Sets the port of the Jackett server in the ServerConfig.json, needs to match the **exposed port** in the Dockerfile  |`WEBUI_PORT=9117`|`9117`|
|`HEALTH_CHECK_HOST`| No |This is the host or IP that the healthcheck script will use to check an active connection|`HEALTH_CHECK_HOST=one.one.one.one`|`one.one.one.one`|
|`HEALTH_CHECK_INTERVAL`| No |This is the time in seconds that the container waits to see if the internet connection still works (check if VPN died)|`HEALTH_CHECK_INTERVAL=300`|`300`|
|`HEALTH_CHECK_SILENT`| No |Set to `1` to supress the 'Network is up' message. Defaults to `1` if unset.|`HEALTH_CHECK_SILENT=1`|`1`|
|`ADDITIONAL_PORTS`| No |Adding a comma delimited list of ports will allow these ports via the iptables script.|`ADDITIONAL_PORTS=1234,8112`||

## Volumes
| Volume | Required | Function | Example |
|----------|----------|----------|----------|
| `config` | Yes | Jackett and OpenVPN config files | `/your/config/path/:/config`|
| `blackhole` | No | Default blackhole path for saving magnet links | `/your/blackhole/path/:/blackhole`|

## Ports
| Port | Proto | Required | Function | Example |
|----------|----------|----------|----------|----------|
| `9117` | TCP | Yes | Jackett WebUI | `9117:9117`|

# Access the WebUI
Access http://IPADDRESS:PORT from a browser on the same network. (for example: http://192.168.0.90:9117)

## Default Info
API Keys are randomly generated the first time that Jackett starts up. There is no Web UI password configured. This can be done manually from the Web UI.

| Credential | Default Value |
|----------|----------|
|`API Key`| Randomly generated |
|`WebUI Password`| No password |

# How to use OpenVPN
The container will fail to boot if `VPN_ENABLED` is set and there is no valid .ovpn file present in the /config/openvpn directory. Drop a .ovpn file from your VPN provider into /config/openvpn (if necessary with additional files like certificates) and start the container again. You may need to edit the ovpn configuration file to load your VPN credentials from a file by setting `auth-user-pass`.

**Note:** The script will use the first ovpn file it finds in the /config/openvpn directory. Adding multiple ovpn files will not start multiple VPN connections.

## Example auth-user-pass option for .ovpn files
`auth-user-pass credentials.conf`

## Example credentials.conf
```
username
password
```

## PUID/PGID
User ID (PUID) and Group ID (PGID) can be found by issuing the following command for the user you want to run the container as:

```
id <username>
```

# Issues
If you are having issues with this container please submit an issue on GitHub.
Please provide logs, Docker version and other information that can simplify reproducing the issue.
If possible, always use the most up to date version of Docker, you operating system, kernel and the container itself. Support is always a best-effort basis.