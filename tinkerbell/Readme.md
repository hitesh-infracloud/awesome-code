## What is Bare Metal?
It is a piece of hardware that can range from being as small as a raspberry pie to being as complex as a data center.
But nevertheless it should have below basic supports:
1. n/w -> support for IPMI
2. Storage
3. Boot environment -> support for iPXE (essential for n/w booting)

## Network booting
1. PXE/iPXE
   1. PXE - an executable environment.
   2. Was introduced in 1998.
   3. It is basically a minimal environment to install another fully functional OS or something similar to OS.
   4. iPXE - extended version of PXE and is open source.
2. DHCP:
   1. Provides IP dynamically.
   2. Was introduced in 1993.
   3. Role is to give n/w configuration to a device that needs it.
3. TFTP:
   1. Provides initial files system.
   2. Simple tech to transmit data. 
   3. Used in conjunction with DHCP as DHCP response from a DHCP server contains TFTP filepath and TFTP server address. So that the booting server can access and download them.
4. NFS -> Required if there is no storage in the hardware.
5. Boot flow steps:
   1. Power is applied to the server.
   2. Server will complete its POST (power on self test).
   3. BIOS is typically configured to start reading from a persistent disk.
   4. But is disk is not present or if disk does not have anything to execute. It looks for something over the network it is connected to.
   5. It sends out a DHCP request by sending out its mac address.
   6. There must be a 'Deployment Server' (that hosts DHCP and TFTP services) that listens for such requests over the n/w.
   7. This 'Deployment Server' provides an IP address, boot server address and PXE/iPXE filename.
   8. The booting server connects to the boot server address and downloads the PXE/iPXE file.
   9. On executing the PXE/iPXE file a minimum environment is available in the booting server.
   10. The booting server again sends a DHCP request for n/w configuration.
   11. The deployment server now sends address of a fully fledged OS as it recognises that booting server already has a minimum environment running int it from the DHCP request.
   12. Now the OS is installed in the booting server.
   13. So basically above steps are being handled by 'Boots' service of Tinkerbell.

## Bare metal UseCases
1. Existing infrastructure service - A lot of organizations already have data centers (before cloud boost) and the wish to leverage them.
2. Data security - most of the time in cloud infra we are unaware where our data resides. So organizations like finance institutions would like to protect their data from theft. And it starts with the knowledge of where the data resides.
3. Latency - there will always be limitation when it comes to performance while using cloud. But in certain cases where real time data is of absolute importance, latency cannot be a tradeoff.
4. Consistent and predictable performance.

## Bare metal Challenges
1. Difficult to manage large infra.
2. Different CPUs like Intel, ARM, different distros.
3. Increase of control comes with increase in complexity.

## Tinkerbell - Rescuer
1. It is a solution for 'Bare metal challenges'.
2. Automated bare metal provisioning.
3. It comprises 5 components:
   1. Tink: Provisioning and workflow engine.
   2. Boots: DHCP and TFTP server.
   3. Hegel: Metadata service.
   4. OSIE: OS installation environment. It is not essentially a service.
   5. PBnJ: Used to control the Worker Server (start, stop, reboot like functionalities).
4. Tink:
   1. The Tink has three major components:
      1. tink-cli: used by users to communicate with tink-server and the communication happens via gRPC. Analogous to how kubectl communicates with kube-api-server.
      2. tink-server: it essentially handles communications from tink-cli as well as tink-worker (running on worker VMs) with the help of 'Boots', 'Hegel' and 'OSIE'.
      3. tink-worker: this runs on worker VMs and does activities like: start the worker, connects with tink-server, fetch workflow and tasks, execute the tasks.
   2. Apart from 'Hegel', 'Boots' and 'OSIE', tink-server has below components as well:
      1. DB: postgres flavor. Stores and maintains workflows, templates and hardware info.
      2. Registry: a private docker registry that is a collection of images used as actions as a part of workflow. This is useful for secure environments without access to internet.
      3. nginx: This serves required boot files.
5. Tink-Cli
   1. To get inside tink-cli:
      1. `vagrant ssh provisioner`
      2. `docker exec -it <tink-cli container name or id> sh`
   2. `tink hardware get` -> get list of hardware configured. [tink_hardware_list.png](https://github.com/hitesh-infracloud/awesome-code/tree/master/tinkerbell/dist/tink_hardware_list.png)
   3. `tink hardware mac <mac address> --details` -> get details of a particular h/w
      ```
      {
       "metadata": {
       "instance": {},
       "facility": {
           "facility_code": "onprem"
        }
      },
      "network": {
          "interfaces": [
              {
                 "dhcp": {
                     "mac": "98:03:9b:89:d7:aa",
                     "hostname": "localhost",
                     "arch": "x86_64",
                     "ip": {
                         "address": "192.168.1.6",
                         "netmask": "255.255.255.248",
                         "gateway": "192.168.1.1"
                     }
                 },
                 "netboot": {
                     "allow_pxe": true,
                      "allow_workflow": true
                 }
              }
          ]
      },
      "id": "f9f56dff-098a-4c5f-a51c-19ad35de85d4"
      }
      ```
      [get_hardware_by_hardware-id.png](https://github.com/hitesh-infracloud/awesome-code/tree/master/tinkerbell/dist/get_hardware_by_hardware-id.png)
      [get_hardware_by_mac.png](https://github.com/hitesh-infracloud/awesome-code/tree/master/tinkerbell/dist/get_hardware_by_mac.png)
   4. `tink template get` -> to get list of templates. [get_all_templates.png](https://github.com/hitesh-infracloud/awesome-code/tree/master/tinkerbell/dist/get_all_templates.png)
   5. `tink template get <template id>`. [get_template_details.png](https://github.com/hitesh-infracloud/awesome-code/tree/master/tinkerbell/dist/get_template_details.png)
   6. `tink workflow get` [get_workflows.png](https://github.com/hitesh-infracloud/awesome-code/tree/master/tinkerbell/dist/get_workflows.png)
   7. `tink workflow create -t <template id> -r '{"device_id":"mac address"}'`
      ```
      version: "0.1"
      name: ubuntu_provisioning
      global_timeout: 6000
      tasks:
      - name: "os-installation"
        worker: ""
        volumes:
        - /dev:/dev
        - /dev/console:/dev/console
        - /lib/firmware:/lib/firmware:ro
        environment:
           MIRROR_HOST: <MIRROR_HOST_IP>
        actions:
        - name: "disk-wipe"
          image: disk-wipe
          timeout: 90
        - name: "disk-partition"
          image: disk-partition
          timeout: 600
          environment:
            MIRROR_HOST: <MIRROR_HOST_IP>
          volumes:
          - /statedir:/statedir
       - name: "install-root-fs"
         image: install-root-fs
         timeout: 600
       - name: "install-grub"
         image: install-grub
         timeout: 600
         volumes:
         - /statedir:/statedir
      ```
   8. `tink workflow gwt <workflow id>`. [get_workflow_by_id.png](https://github.com/hitesh-infracloud/awesome-code/tree/master/tinkerbell/dist/get_workflow_by_id.png)