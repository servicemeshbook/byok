## Set Network Address VMware Workstation

Go to Edit ⇨  Virtual Network Editor

Select `vmnet8` and if necessary hit `Change Settings` to make changes.

Make sue that the subnet IP for `vmnet8` is set to `192.168.142.0`.

From VM, ping VMware gateway address `192.168.142.2`. If ping succeeds, you should be able to ping `ping -c4 google.com`

If still, you can't ping google, make sure that the `/etc/resolv.conf` in the VM is set to `192.168.142.2`

Example:

```
nameserver 192.168.142.2
domain servicemesh.local
search servicemesh.local
```

## Set Network Address in VMware Player

If you are not using VMware Workstation (it costs money), you can download free VMware Player from vmware.com and install it on your Windows.

> There is no VMware Player for Mac. The only option is to use VMware Fusion.

Please note: You can only run one VM per hosts using VMware Player (free version).
