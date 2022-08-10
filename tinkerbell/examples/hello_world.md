## Pre-Requisites
1. Provisioner up and running (preferably in local vagrant).
2. Worker that is not up yet or can be restarted.

## Push an image to 'registry' component of provisioner
1. `docker pull hello-world`
2. `docker tag hello-world <registry host>/hello-world`
   1. registry host for us was 'https://192.168.60.4:443'
   2. also we had to do below steps to skip certification verify
      1. create a file at /etc/docker - daemon.json
      2. copy below contents:
         ```
         {
           "insecure-registries" : ["https://192.168.60.4:443"]
         }
         ```
      3. pkill dockerd
      4. sudo dockerd > /dev/null 2>&1 &
3. `docker login` - enter username and password
4. `docker push <registry host>/hello-world`
5. verify if registry service has the newly pushed hello-world image
   1. `docker exec -it <registry container id> sh`
   2. `ls /var/lib/registry/docker/registry/v2/repositories/` - this should have a directory 'hello-world'

## Work with template
1. Exec into tink-cli container running inside provisioner: `docker exec -it <tink cli container id>`
2. Create a template file with below contents:
    ```
    version: "0.1"
    name: hello_world_workflow
    global_timeout: 600
    tasks:
    - name: "hello world"
      worker: "{{.device_1}}"
      actions:
      - name: "hello_world"
      image: hello-world
      timeout: 60
   ```
3. Create template command: `tink template create --file "./hello-world.tmpl" -r '{"device_1":"08:00:27:9e:f5:3a"}'`
4. Push hardware details if already not done: `tink hardware push --file <hardware.json>`. File should be in the lines of [sandbox hardware.json](https://github.com/tinkerbell/sandbox/blob/main/deploy/compose/create-tink-records/manifests/hardware/hardware.json)
5. Create workflow: `tink workflow create -t <template_id> -r '{"device_1":"08:00:27:9e:f5:3a"}'`. Use template id from #3.
6. Restart the worker VM from virtual box and check events: `tink workflow events <workflow id>`. Use workflow id from #5.

