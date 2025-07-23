# LenovoArchLinux
Installation notes for Arch on Lenovo Pro 7 16IRX9H

## Use Wireless for Install

Enter interactive prompt
```
$ iwctl
```

List devices
```
[iwd]# device list
```

May have to turn on the device/adapter
```
[iwd]# device <device name> set-property Powered on
[iwd]# adapter <adapter name> set-property Powered on
```

Initiate a scan (will not output anything)
```
[iwd]# station <device name> scan
```

List networks
```
[iwd]# station <device name> get-networks
```

Connect to network (quote SSID if contains spaces)
```
[iwd]# station <device name> connect <SSID>
```

## Prevent speakers from turning off randomly due to power save

Credit to edkirin on the [Manjaro Forums](https://forum.manjaro.org/t/finally-got-sound-working-on-lenovo-legion-pro-7-16arx8h/164447) and others on the [Arch Forums](https://bbs.archlinux.org/viewtopic.php?id=304863)

Create a script ```/usr/local/bin/tas2781-fix``` with the following contents:
```
#!/bin/sh

if [ "$(id -u)" -ne 0 ]; then
  printf "You must run this script as root.\n"
  exit 1
fi

POWER_SAVE_PATH="/sys/module/snd_hda_intel/parameters/power_save"
POWER_CONTROL_PATH="/sys/bus/i2c/drivers/tas2781-hda/i2c-TIAS2781:00/power/control"

check_paths() {
  [ -e "$POWER_SAVE_PATH" ] && [ -e "$POWER_CONTROL_PATH" ]
}

while ! check_paths; do
  sleep 1
done

# Disable snd_hda_intel power saving
printf "0" > "$POWER_SAVE_PATH"

# Disable runtime suspend/resume for tas2781
printf "on" > "$POWER_CONTROL_PATH"
```

Make the script executable
```
chmod +x /usr/local/bin/tas2781-fix
```

Now make a systemd unit ```/etc/systemd/system/tas2781-fix.service```:
```
[Unit]
Description=Run the tas2781-fix script after the relevant sysfs paths become available

[Service]
Type=oneshot
ExecStart=/usr/local/bin/tas2781-fix
RemainAfterExit=true
TimeoutSec=60

[Install]
WantedBy=multi-user.target
```

Create the power_save and control files initialized to turn off power save
```
echo 0 | sudo tee /sys/module/snd_hda_intel/parameters/power_save
echo on | sudo tee /sys/bus/i2c/drivers/tas2781-hda/i2c-TIAS2781:00/power/control
```

Enable the unit and restart
```
systemctl daemon-reload
systemctl enable tas2781-fix.service
```

Reboot
