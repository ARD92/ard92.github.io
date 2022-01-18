# Proxy HTTP and HTTPS Connections 

sometimes you may want to proxy connections from your ubuntu server to another machine. lets say you want to proxy connections from a device (a server in your lab) to your laptop so that apt-packages can resolve. one can do that by installing squidman on MAC to run it as a proxy server. follow the below steps to try it out! 

##  Proxy the connections from ubuntu server on VPN 
Install Squidman to run MAC as a proxy server . you can download from [here](https://squidman.net/squidman/)

- once squidman is installed, go to the `preferences` tab .
- http port is by default 8080. This can be left alone unless you have a different application using the same port
- click on tab `clients` and add the subnet of the interface where traffic would be arriving from the client device. 
    - for example: my lab server is connected over VPN so , i will look at the IP allocated to the interface using `ifconfig` 
    ```
    utun2: flags=8050<POINTOPOINT,RUNNING,MULTICAST> mtu 1400
	inet w.x.y.z --> w.x.y.z netmask 0xffffffff
    ```
    - in case there is no VPN , look at the wifi/wired interface where you receive the IP of your device 
- add the IP range in the `clients` tab using `new` 

## Export env variables to proxy http and https connections to the laptop/new proxy server
Ensure, the client device (lab server) can reach the proxy server IP (your VPN interface or your wired/wireless interface in case of no VPN) and export the ENV variables.
```
HTTPS_PROXY="http://<IP>:<PORT>"
HTTP_PROXY="http://<IP>:<PORT>"
```
Here the `IP` is `w.x.y.z` which is the IP we retrieved earlier and
`PORT` is `8080` which is the default port used in squidman 

## Edit apt configs
Edit/add the apt.conf to reflect the changes. 

Note: In case the configs dont exist, create the file

```
root@ubuntu:/ more /etc/apt/apt.conf
Acquire::socks::proxy "socks5://w.x.y.z:8080";
```
Note that w.x.y.z is the interface IP (VPN or wired/wireless) of the proxy server.

Once the above is done, try doing a `apt update` or `apt install <PACKAGE NAME>` to see if every http/https request for apt is proxyd though your new proxy server
