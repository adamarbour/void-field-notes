# Wifi
1. Start bash instead of sh
```bash
bash
PS1='(live) #'
```
2. Check and make sure the devices are not blocked
```bash
rfkill
rfkill unblock wifi # Nothing was blocked but this is an example.
```
2. Get the wifi device name
```bash
ip link # w1p1s0
```
3. Connect to wifi
```bash
wpa_supplicant -B -i interface -c <(wpa_passphrase MYSSID passphrase)
```
4. Bring the interface up
```bash
ip link set dev wlp1s0 up
```
