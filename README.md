itnaw-local
===========

This is a docker-compose setup to spin up the resources for the "Introduction To Network Automation Workshop" (`itnaw`) so you can lab your face off whenever you want (with some caveats...)!


# Requirements

This must run on a machine that has Docker (of course), but also that supports nested virtualization (for the containerlab/boxen VM images). If nested virtualizaiton is unavailable, the lab will still start, however only the Arista network devices will be launched (no CSR/N9Kv/PAN devices).

The machine running this needs a lot of resources! The VM in the "full" lab setup is a 8 core, 32gb virtual machine -- the PAN/N9Kv resources in particular are rather... greedy! This setup likely requires *more* resources as this includes containers for the k3s cluster and all the resources that run on it.

You need images for the network resources. Well... you need images for all the resources, but you can just auto-pull most of them since they are public/available via docker hub or ghcr. For the *network* devices, however, those images are *not* public/available. You can acquire the Arista cEOS image by simply creating an Arista account and downloading the tar file and importing the image. The CSR1000v/N9Kv/PAN images are another story! You must acquire the appropriate qcow2 disk images of these, and then generate the image that [containerlab](https://github.com/srl-labs/containerlab) can use with the [boxen](https://github.com/carlmontanari/boxen) utility.


# Accessing K3s Resources

In the "full" lab, all the resources you would access live in GKE, in this "local" setup, they all live in k3s. Despite being in k3s and deployed locally, the deployments are basically the same -- including the nginx ingress. The nginx ingress allows access to the resources based on the http path you try to access. In order to let you point to "regexr.compubotz.com" like you would for the "full" lab, you simply need to lie to your machine to redirect that to your local host. Add the following entries to /etc/hosts:

```
127.0.0.1 regexr.compubotz.com
127.0.0.1 peeringdb.compubotz.com
127.0.0.1 student-jenkins.compubotz.com
127.0.0.1 student-netbox.compubotz.com
```

To avoid exposing these resources on the "real" ports 80/443, the k3s/docker setup exposes them on 8080/8443 -- so once things are fired up and ready, and your /etc/hosts entries are set, you can simply web to any of the URLs above on 8080 or 8443 like: "https://peeringdb.compubotz.com:8443" or "http://peeringdb.compubotz.com:8080".


# Accessing Network Resources

In the "full" lab, you can simply access the network devices via their names "leaf1", "leaf2", etc.. Containerlab sorts out all the networking bits by putting the management interfaces into the docker bridge. For this "local" setup, you may not have that luxury (ex: running on Darwin because of the docker VM), to work around this, the management ports are exposed via NAT. The following table shows all exposed ports for these devices:

| Device | SSH   | Telnet | HTTP  | HTTPS | NETCONF |
|--------|-------|--------|-------|-------|---------|
| spine1 | 20022 | 20023  | 20080 | 20443 | 20830   |
| leaf1  | 21022 | 21023  | 21080 | 21443 | 21830   |
| leaf2  | 22022 | 22023  | 22080 | 22443 | 22830   |
| leaf3  | 23022 | 23023  | 23080 | 23443 | 23830   |
| fw1    | 24022 | 24023  | 24080 | 24443 | 24830   |
| rtr1   | 25022 | 25023  | 25080 | 25443 | 25830   |

So you can SSH to the "fw1" instance like: "ssh -l admin -p 24022 localhost", or access HTTPS on leaf3 like: "https://localhost:23443".


# Accessing Hoppscotch

Hoppscotch is just running as a service in the docker-compose file, so it just runs normally -- you can web to it like: "http://localhost:8001".


# Running Things

Job one is to update the `clab.env` file with the appropriate container images for the networking devices. This environment file gets passed to the "clab-launcher" service which tells containerlab what images to use for each of the NOSs -- once again, you need to provide these images yourself, the images in the provided clab.env are just examples.

Simply run `docker compose up` / `docker compose up -d` (detached)! 

You can tail the output of the "clab-launcher" and "k3s-launcher" or if not using detached just watch the console. The startup process takes quite a while as the k3s cluster needs to get launched, then manifests deployed, and of course the network devices take a while to boot up as well.

Successful output will look something like this:

```
docker compose up
[+] Running 1/1
 ⠿ hoppscotch Pulled                                                                                             0.6s
[+] Running 4/4
 ⠿ Network itnaw                          Created                                                                0.0s
 ⠿ Container itnaw-local-clab-launcher-1  Created                                                                0.1s
 ⠿ Container itnaw-local-hoppscotch-1     Created                                                                0.1s
 ⠿ Container itnaw-local-k3s-launcher-1   Created                                                                0.1s
Attaching to itnaw-local-clab-launcher-1, itnaw-local-hoppscotch-1, itnaw-local-k3s-launcher-1
itnaw-local-k3s-launcher-1   | starting itnaw kubernetes resources
itnaw-local-k3s-launcher-1   | starting k3s cluster with command '['k3d', 'cluster', 'create', 'itnaw', '--servers', '3', '--k3s-arg', '--disable=traefik@server:0;server:1;server:2', '-p', '8080:80@loadbalancer', '-p', '8443:443@loadbalancer', '--token', 'superSecretToken', '--wait']'
itnaw-local-clab-launcher-1  | checking if nested virtualization is available...
itnaw-local-clab-launcher-1  | kvm is available!
itnaw-local-clab-launcher-1  | starting lab
itnaw-local-clab-launcher-1  | starting with command ['containerlab', '-t', '/clab-full.yaml', 'deploy']
itnaw-local-hoppscotch-1     |
itnaw-local-hoppscotch-1     | > hoppscotch-app@2.0.0 start /app
itnaw-local-hoppscotch-1     | > pnpm -r do-prod-start
itnaw-local-hoppscotch-1     |
itnaw-local-hoppscotch-1     | Scope: all 3 workspace projects
itnaw-local-hoppscotch-1     | packages/hoppscotch-app do-prod-start$ pnpm run start
itnaw-local-hoppscotch-1     | packages/hoppscotch-app do-prod-start: > hoppscotch-app@2.0.0 start /app/packages/hoppscotch-app
itnaw-local-hoppscotch-1     | packages/hoppscotch-app do-prod-start: > nuxt start
itnaw-local-hoppscotch-1     | packages/hoppscotch-app do-prod-start: ℹ Listening on: http://172.20.0.101:3000/
itnaw-local-hoppscotch-1     | packages/hoppscotch-app do-prod-start: ℹ Serving static application from dist/
itnaw-local-k3s-launcher-1   | k3s cluster started successfully!
itnaw-local-k3s-launcher-1   | deploying kubernetes manifests...
itnaw-local-k3s-launcher-1   | deploying manifests from manifests/1-base directory...
itnaw-local-k3s-launcher-1   | deploying manifests from manifests/2-ingress directory...
itnaw-local-k3s-launcher-1   | sleeping a bit to let ingress do its thing
itnaw-local-k3s-launcher-1   | deploying manifests from manifests/3-apps directory...
itnaw-local-k3s-launcher-1   | deploying manifests from manifests/4-student directory...
itnaw-local-k3s-launcher-1   | manifests deployed!
```

You can check the state of the network devices with `docker ps`:

```
docker ps
CONTAINER ID   IMAGE                                             COMMAND                  CREATED         STATUS                     PORTS                                                                                                                                                                                                                                          NAMES
748a7f9100e5   ghcr.io/compunetinc/itnaw-vr-n9kv-boxen:9.2.4     "boxen package-start…"   4 minutes ago   Up 4 minutes (unhealthy)   4001/tcp, 161/udp, 5001/tcp, 0.0.0.0:23022->22/tcp, :::23022->22/tcp, 0.0.0.0:23023->23/tcp, :::23023->23/tcp, 0.0.0.0:23080->80/tcp, :::23080->80/tcp, 0.0.0.0:23443->443/tcp, :::23443->443/tcp, 0.0.0.0:23830->830/tcp, :::23830->830/tcp   clab-itnaw-leaf3
```

The "status" column will eventually show "healthy" for the "vr-X-boxen" VM images (CSR/N9Kv/PAN devices). At that point, everything should be good to go!

