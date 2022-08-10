## Pre-Requisites
1. Provisioner up and running (preferably in local vagrant).
2. Worker that is not up yet or can be restarted.

## Create and Push FlatCar docker image to tinkerbell registry
1. SSH to provisioner: `vagrant ssh provisioner`
2. Create a directory: `mkdir flatcar && cd flatcar`
3. Create a Dockerfile with below contents: `touch Dockerfile`
    ```
    FROM ubuntu
    
    RUN apt update && apt install -y udev gpg wget
    
    RUN wget https://raw.githubusercontent.com/flatcar-linux/init/flatcar-master/bin/flatcar-install -O /usr/local/bin/flatcar-install && \
        chmod +x /usr/local/bin/flatcar-install
    ```
4. Build a docker image out of the above: `docker build -t <registry host>/flatcar-install .`
5. Push the image to registry: `docker push <registry host>/flatcar-install`
6. Registry host for us was: 192.168.60.4:443
7. Verify if image got pushed into the registry:
   1. `docker exec -it <registry container id> sh`
   2. `ls /var/lib/registry/docker/registry/v2/repositories/` - this should have a directory 'flatcar-install'

## Prepare local mirror for installation of flatcar-install linux image
1. Creating mirror:
   ```
   mkdir /var/tinkerbell/nginx/misc/flatcar
   cd /var/tinkerbell/nginx/misc/flatcar
   wget https://github.com/kinvolk/flatcar-release-mirror/raw/master/flatcar-release-mirror.sh && chmod +x flatcar-release-mirror.sh
   ```
2. Run the mirror script: (It will download all Flatcar images above version 2344 only from stable channel. This currently requires downloading around 1.1 GB of data)
   ```
   ./flatcar-release-mirror.sh --only-files 'flatcar_production_image\|version' --above-version 2344 --logfile /dev/stderr --channels stable
   ```

## Create template
1. exec into tink-cli: `docker exex -it <tink cli conatiner id> sh`
2. Create template file (flatcar-install.tmpl)
   ```
   version: "0.1"
   name: flatcar-install
   global_timeout: 1800
   tasks:
     - name: "flatcar-install"
       worker: "{{.machine1}}"
       volumes:
         - /dev:/dev
         - /statedir:/statedir
       actions:
         - name: "dump-ignition"
           image: flatcar-install
           command:
             - sh
             - -c
             - echo '{{.ignition}}' | base64 -d > /statedir/ignition.json
         - name: "flatcar-install"
           image: flatcar-install
           command:
             - /usr/local/bin/flatcar-install
             - -s
             - -i
             - /statedir/ignition.json
   ```

3. The command section of the template can be change to below to use previously created local mirror:
   ```
   - -b
   - http://192.168.60.3/misc/flatcar/stable/amd64-usr
   ```
4. `tink template create -n flatcar-install -p /tmp/flatcar-install.tmpl`

where 192.168.60.3 is the nginx url obtained from 'TINKERBELL_NGINX_IP' of setup.sh

## Ignition configuration
The recommended way of configuring flatcar is via ignition file during installation.
1. Create a config.yaml as below:
   ```
   passwd:
     users:
       - name: core
         ssh_authorized_keys:
           - YOUR_SSH_KEY
   ```
2. encode the configuration: `ct -in-file config.yaml | base64 -w0; echo


## Create workflow
`tink workflow create -t TEMPLATE_ID -r '{"machine1": "<mac address>", "ignition": "<encrypted ignition file sha>"}'`

## Provisioning
1. Reboot the worker VM and observe:
   `tink workflow state <WORKFLOW_ID>`
   `tink workflow events <WORKFLOW_ID>`



