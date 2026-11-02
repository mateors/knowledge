# Server essential commands

## How do i know my linux os restart required?
To check if your Ubuntu or Debian-based Linux system requires a reboot, look for the presence of the /var/run/reboot-required file.

### Check if the reboot file exists:
```
[ -f /var/run/reboot-required ] && echo "Reboot required" || echo "No reboot needed"
```

or 

```
root@mail:~# cat /var/run/reboot-required
*** System restart required ***
```

### See which specific packages triggered the reboot request:
```bash
cat /var/run/reboot-required.pkgs
```

Expected output:
```
root@mail:~# cat /var/run/reboot-required.pkgs
libc6
linux-image-5.15.0-198-generic
linux-base
```
