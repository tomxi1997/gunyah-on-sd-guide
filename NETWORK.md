# Networking on cromvm on android
[The official guide for crosvm](https://crosvm.dev/book/devices/net.html) includes instructions for setting up a NAT network, but those are intended for Desktop Linux and require some additional configuration to work on Android.

https://github.com/polygraphene/gunyah-on-sd-guide/blob/9a475d7bba51cbc8fde7c22abac35cccb40a58e0/run-crosvm-net.sh#L1-L35

Copy and paste them or download with wget like this:

```
# wget https://raw.githubusercontent.com/polygraphene/gunyah-on-sd-guide/refs/heads/main/run-crosvm-net.sh
```

Then edit /etc/netplan/90-default.yaml in the VM like this:
```yaml
network:
    version: 2
    ethernets:
        all-en:
            match:
                name: en*
            dhcp4: false

            addresses:
              - 192.168.8.2/24
            routes:
              - to: default
                via: 192.168.8.1
            nameservers:
                  addresses: [8.8.8.8]
            dhcp6: true
            dhcp6-overrides:
                use-domains: true
        all-eth:
            match:
                name: eth*
            dhcp4: true
            dhcp4-overrides:
                use-domains: true
            dhcp6: true
            dhcp6-overrides:
                use-domains: true
```

Then run
```sh
# netplan apply
# ping www.google.com
...
```
