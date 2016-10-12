# Setting up an Ethereum Parity node on a QNAP NAS with Docker [Draft]

_Note that this remains something of a draft. Currently my node is struggling to stay up due to a network attack that is drastically increasing memory consumption. If/when things are stabilised, I'll be able to revise this more._

## Introduction

By [popular](https://gitter.im/ethcore/parity?at=57fac0fb70fcb5db0c42167b) [demand](https://gitter.im/ethcore/parity?at=57f6c5d6d6251fd1269a659c), I'm documenting my adventures in setting up an Ethereum Parity node on a home NAS box. For an encore I linked it to the [Ethereum Network Status page](https://ethstats.net/).

Since I'm exclusively using Docker in the below, all this should pretty much translate directly across to other Docker-supporting platforms, at least the command-line bits, but no promises.

There seems to a dearth of straightforward "how-to"s on this kind of stuff, although I did find[this] (https://medium.com/@preitsma/setting-up-a-parity-ethereum-node-in-docker-and-connect-safely-f881faa17686) very helpful in getting started. Anyway, I hope this helps. 

## Disclaimer

All this is 100% new to me. At the time of first setting this up, I'd had the NAS box for less than one week, I had absolutely zero knowledge of Docker, I'd run a Parity node on my laptop, but had only the haziest idea of how it was working... so proceed with appropriate caution. There may well be better ways to do this: if you know of one, I'd be glad to hear about it.

## The Hardware

I'm running this on a [QNAP TS-253A](https://www.qnap.com/en-uk/product/model.php?II=211&ref=product_overview) NAS with 4GB memory. It has built-in support for Docker which is what makes all of this relatively straightforward, after getting one's head round Docker, that is.

As for networking, the kit is sitting at the end of a domestic home-plug network connected to the outside world by an approx. 18Mbps copper ADSL link. This is not high-end stuff. Even so, the node seems to be holding its own in comparison with others on ethstats.net (at least in the absence of the memory-intensive network attack).

# Set-up of the Parity node

## Initial set-up

I started with the QNAP _Container Station_ GUI, but quickly dropped to a command line to do most things. The GUI remains useful for quickly stopping and restarting containers, and for setting resources limits.

### Via the GUI

Using the QNAP GUI in a Web browser I installed _Container Station_ which handles installing and monitoring Docker images and containers.

For my very first install of Parity I used the QNAP GUI app. In _Container Station_, select _Create Container_ and type "parity" in the search box. Available images will show up on the _Docker Hub_ tab. Next to "ethcore/parity" click "Create". It will ask you to select which version; select "latest". After acknowledging the pop-up disclaimer, you can then click "Create" on the _Create Container_ dialogue. Parity should now start running!

Back in the _Container Station_ Overview tab you can click on the running container and see what it's up to. Clicking on the arrows next to _Console_ will open up a new Web browser tab containing fuller output from the container's console. You should see Parity announcing its version, its ehash and then starting to sync.  My first sync was done overnight. The container hung up at some point near the end so I just restarted it and it was able to finish fine.

### Via the command line

As an alternative to the above, you can also `ssh` into the QNAP as administrator and just type
```
docker pull ethcore/parity:latest
docker run -d --name my_parity_latest ethcore/parity:latest
```

TODO - check the run command.

## Extracting the blockchain folder

At some point you're going to want to update the Docker image. This happened a lot for me: my initial set up was during the Great [Ethereum Week of Network Attacks](https://blog.ethereum.org/2016/09/22/ethereum-network-currently-undergoing-dos-attack/), and there were several new ethcore/parity:latest image versions released every day. I soon realised that if I just downloaded and ran a new image, I'd have to begin syncing all over again from scratch. Not an appealing prospect.

To avoid this, it is necessary to store the blockchain data outside the Parity container.

### Copying the blockchain out of the container

This requires running a command line on the NAS. I use `ssh` to log in as admin, but there is also an app that provides a command line in a browser window if you like. TODO - what is this app's name?

First, use `docker cp` to copy the blockchain data you've already downloaded out of the container onto the NAS's own filesystem. I stopped the Parity container to do this, but I believe this may not be necessary.

```
docker cp my_parity_latest:/root/.parity /share/homes/admin/parity
```

This copies the whole of the container's _.parity_ directory, which includes the blockchain database directory, into a local directory on the NAS called _parity_.

Note that _my\_parity\_latest_ is the name of my Parity container. Use your name here if you called it something different.

### Mounting the blockchain directory back on to containers

Now when we run a Parity image, we can attach the NAS's copy of the blockchain to the container and Parity will start syncing from there rather than right from the beginning. This is done with the `-v` flag to `docker run`:

```
docker run -d --name my_parity_latest -v /share/homes/admin/parity:/root/.parity etchcore/parity:latest
```

## Using a _docker-compose.yml_ file

It turns out that a convenient way of updating containers and storing configuration info is to use something called a [compose file](https://docs.docker.com/compose/overview/). These are called _docker-compose.yml_.  One of the advantages is that Docker Compose can automatically check for newer image files in the repository and download them accordingly.

Here's a _docker-compose.yml_ file that implements the configurations we've done so far.

```
ethereum-node:
  image: ethcore/parity:latest
  container_name: my_parity_latest
  volumes:
    - /share/homes/admin/parity:/root/.parity
```

Now all we need to do to update to the newest version of ethcore/parity:latest is the following (make sure you are in the same directory as the compose file). If there is no update it won't do anything. If you've edited the compose file, or if the first command has found and downloaded a new image file, it will restart the container.

```
docker-compose pull
docker-compose up -d
```

### Network port assignments

Note that I found it was not necessary to add any network port mappings to the file. Docker seems to work it out. If you want to open the RPC port, then you need to do some mapping for port 8545, but that's a whole other story that I'm not going to cover here.

## Resource limits

You probably don't want Parity completely taking over your NAS box; it seems pretty happy to eat up however much memory you throw at it. You can limit the memory and CPU resources in the _Container Station_ GUI by clicking on the Parity container name (_my\_parity\_latest_ for me) and then on "Settings" in the top right. In general it was running fine with 1024MB, but see below for updates about the recent memory-hogging network attacks.

You don't have to restart the container (uncheck the box at the bottom), but it seems a good idea to do so if you are reducing the memory limit.

You can check the actual memory used by the container from the command line. Do a `cd` to `/sys/fs/cgroup/memory/docker/` and then `ls`. You will see container IDs listed - long hexadecimal names - only one if you're just running Parity.

To see the memory actually in use by the container, substitute the container ID in the following.
```
cat CONTAINER_ID/memory.usage_in_bytes
```

To check that the memory limit has been applied, do this.
```
cat CONTAINER_ID/memory.limit_in_bytes
```

You can also try running `docker stats` on a container, though I've had odd results when running multiple containers. More info [here](https://www.datadoghq.com/blog/how-to-collect-docker-metrics/#toc-stats-command6).

Note that every time the container is updated the resource limits need to be reset manually. It's on my _Todo_ list to work out how to do this in the _docker-compose.yml_ file.

# Linking with _ethstats.net_

## Introduction

Now you've got your Parity node running, you want it to show up on the very funky https://ethstats.net/ dashboard, right? Unfortunately, this is not entirely straightforward, but I'll tell you what's working for me.

The good news is that a docker-based solution is available from the GitHub [eth-net-intelligence-api](https://github.com/cubedro/eth-net-intelligence-api) repository.

## Build the _ethnetintel_ docker image

Being a total Docker newbie, this initially had me scratching my head, but turned out to be pretty simple in the end.

First, download the [_Docker_ file](https://github.com/cubedro/eth-net-intelligence-api/blob/master/Dockerfile) from the repository.

Now, as per the instructions in the _Docker_ file, you ought to be able to type the following at the NAS command line.

```
docker build -t ethnetintel:latest .
```

This is supposed to build a docker image on the NAS, however it failed for me with an error. Instead, I did this step in a Linux VM I'd installed on the NAS, and then exported the image to the NAS filesystem using `docker save`. It's probably Docker version related. Nonetheless, it is a pain.

## Mounting app.json file and getting the password

Once you have your image, there is a local configuration file required, _app.json_ which needs to be outside the container (similarly to the blockchain data above) so that it survives image updates and container rebuilds. you can [download](https://github.com/cubedro/eth-net-intelligence-api/blob/master/app.json) this too from the repository. I put it in `/share/homes/admin/docker/ethnetintel/app.json` on the NAS filesystem.

The _app.json_ file is shared with/volume mounted on the container by adding the following lines to the _docker-compose.yml_ file for the _ethnetintel_ container, which I'm calling _my\_ethnetintel_.
```
  volumes:
    - /share/homes/admin/docker/ethnetintel/app.json:/home/ethnetintel/eth-net-intelligence-api/app.json
```

Some lines in this file need to be edited for your configuration. It's pretty self-explanatory.

One of the lines sets a WS_SECRET value. This you need to get hold of from someone in the know. I got it by asking nicely on the Parity [Gitter channel](https://gitter.im/ethcore/parity).

## Networking

The Docker file contains some instructions about networking that I didn't really understand at first, but after trying out a few different approaches it seems to be good advice.  Basically, set things up so that only the _my\_ethnetintel_ container has access to the outside world, and make the _my\_parity\_latest_ container share _my\_ethnetintel_'s network.

In the _docker-compose.yml_ file for _my\_parity_latest_ we add this,
```
  net: 'container:my_ethnetintel'
```

This tells the Parity container to use _my\_ethnetintel_'s network so they can talk to each other.

Contrary to other documentation, I found it isn't necessary to make any port mappings between the container and host: Docker seems to work this out by itself.

## Modifications to docker-compose.yml

The docker files for the two services can be combined into one, which has some advantages. See _Putting it all Together_ below.

Nonetheless, you can manage Parity and the Ethereum Net Intelligence tool separately. This means that when I restart the _my\_ethnetintel_ container, I also need to restart _my\_parity\_latest_ because it relies on the other container's networking. However, the Parity container can be stopped, started and updated while letting _my\_ethnetintel_ continue running.

Anyway, the two _docker-compose.yml_ files I am currently using (in separate directories) look like this.

For Parity,
```
ethereum-node:
  image: ethcore/parity:latest
  container_name: my_parity_latest
  volumes:
    - /share/homes/admin/parity:/root/.parity
  net: 'container:my_ethnetintel'
```

For Ethereum Network Intelligence
```
ethnetstats:
  image: ethnetintel:latest
  container_name: my_ethnetintel
  volumes:
    - /share/homes/admin/docker/ethnetintel/app.json:/home/ethnetintel/eth-net-intelligence-api/app.json
```

To get them running I do `docker-compose up -d` for the Ethereum Network Intelligence container, and, once it's running, the same again for the Parity container. Then I have to manually manage the resource settings in the QNAP GUI since I haven't figured out how to set them in the docker compose file yet. See _Todo_ below!

## Resource setting

The _my\_ethnetintel_ container seems fairly light-weight. I've currently allocated it 256MB (it generally uses about a third of this, but has sometimes crashed when set to only 128MB), and I've given it a 20% CPU cap. Both these settings are currently made in the QNAP _Container Station_ GUI and need to be redone every time the container is updated.

# Putting it all together

After rootling around in the Docker Compose documentation for a while, the following reflects my currently recommended workflow.

1. Get the Parity image and extract the blockchain data to a filesystem outside the container as described above.
2. Build the Ethereum Network Status image as described above.
3. Run the following _docker_compose.yml_ file with `docker-compose up -d`, after substituting your own filesystem pathnames in the first part of each of the "volumes" sections.

```
version: '2'
services:
  parity:
    image: ethcore/parity:latest
    volumes:
      - /share/homes/admin/parity:/root/.parity
    depends_on:
      - ethnetintel
    network_mode: 'service:ethnetintel'
  ethnetintel:
    image: ethnetintel
    volumes:
      - /share/homes/admin/docker/ethnetintel/app.json:/home/ethnetintel/eth-net-intelligence-api/app.json
```

This takes care of starting the _ethnetintel_ container before the _parity_ container.

To upgrade Parity if a new Docker image becomes available, just do `docker pull ethcore/parity:latest`. Then repeating step 3 should restart the _parity_ image without unnecessarily restarting _ethnetintel_.

# Issues

## Ethnetinel stops relaying

In the couple of days since I've set this up, the _my\_ethnetintel_ container has twice stopped relaying block information to the Ethereum Network Status page, although Parity has continued running and relaying blocks quite happily. Stopping and restarting the containers (remember to start _my\_ethnetintel_ first!) gets things running again. For some reason, even after an outage of just a few seconds, it can take Parity several minutes to resync, during which it is hammering the disks constantly - swapping I think. I'm guessing that at startup it does some database reorganisation or something which needs a lot of memory. Anyway, don't worry, it'll get there in the end.

## Network memory attacks

In order to counter previous network attacks that used a lot of disk IO, Parity introduced several in-memory caches. Unfortunately, an attacker has now [exploited these](https://www.reddit.com/r/ethereum/comments/570lgm/gavin_wood_the_attacker_is_back_looks_like_hes/) to drive up the memory consumption of nodes in processing certain blocks. My node was crashing even with 2.5GB assigned (above that the NAS gets unhappy); there are reports of nodes needing 3 or more GB.

I'm currently trying a suggestion to resync with the flag `--pruning archive`.  The default is `--pruning fast`. Changing this option sacrifices disk space for lower memory consumption. Disk space? I'm running on a multi-terabyte NAS, so that's not a big problem.

To add this flag (and to use run-time options in general), put the following line in the _docker-container.yml_ file in the _parity_ section.
```
    entrypoint: /build/parity/target/release/parity --pruning archive
```

Now it will resync from scratch. I did copy my old blockchain database first (rename the parity/9xxxx TODO directory) just in case I want to come back to it later.  I'm re-syncing now and will update with the outcome.

# Todo

* set resource limits (CPU/memory) in docker-compose.yml files
