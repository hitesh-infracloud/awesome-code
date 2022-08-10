## Pre-Requisites
1. Provisioner up and running (preferably in local vagrant).
2. Worker that is not up yet or can be restarted.

## Create hardware
1. SSH the provisioner: `vagrant ssh provisioner`
2. Exec into tink-cli: `docker exec -it <tink cli container id> sh`
3. Push a hardware resource: `tink hardware push --file <hardware.json>`. Enter below details into hardware.json
    ```
    {
        "id": "ce2e62ed-826f-4485-a39f-a82bb74338e3",
        "metadata": {
            "facility": {
                "facility_code": "onprem",
                "plan_slug": "c2.medium.x86",
                "plan_version_slug": ""
            },
            "instance": {
                "ipxe_script_url": "https://boot.netboot.xyz/ipxe/netboot.xyz.lkrn",
                "operating_system": {
                    "distro": "custom_ipxe",
                    "slug": "custom_ipxe"
                  }
            },
            "state": ""
        },
        "network": {
            "interfaces": [
                {
                    "dhcp": {
                        "mac": "00:50:56:25:11:0e",
                        "name_servers": [
                            "1.1.1.1",
                            "8.8.8.8"
                        ],
                        "arch": "x86_64",
                        "ip": {
                            "address": "192.168.2.130",
                            "netmask": "255.255.255.0",
                            "gateway": "192.168.2.1"
                        }
                    },
                    "netboot": {
                        "allow_pxe": true,
                        "allow_workflow": true
                    }
                }
            ]
        }
    }   
    ```
4. Reboot the worker VM. It should boot into netboot.xyz menu.

