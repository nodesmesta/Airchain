Check Logs
```
sudo journalctl -u junctiond -f
```
Check Service status
```
sudo systemctl status junctiond
```
Check Info Node
```
junctiond status 2>&1 | jq
```
Start Service
```
sudo systemctl start junctiond
```
Reload Service
```
sudo systemctl start junctiond
```
Stop Service
```
sudo systemctl stop junctiond
```
Restart Service
```
sudo systemctl restart junctiond
```
Enabled Service
```
sudo systemctl enable junctiond
```
Disabled Service
```
sudo systemctl enable junctiond
```
