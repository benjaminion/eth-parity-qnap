# Setting up an Ethereum Parity node on a QNAP NAS with Docker [Draft]

_Note: this is a draft and contains errors, omissions and typos with near certainty_

## Introduction

By [popular](https://gitter.im/ethcore/parity?at=57fac0fb70fcb5db0c42167b) [demand](https://gitter.im/ethcore/parity?at=57f6c5d6d6251fd1269a659c), I'm documenting my adventures in setting up an Ethereum Parity node on a home NAS box. For an encore I linked it to the [Ethereum Network Status page](https://ethstats.net/).

Since I'm exclusively using Docker in the below, then it should pretty much translate directly across to other Docker supporting platforms, at least the command-line bits, but no promises.

There seems to a dearth of straightforward "how-to"s on this kind of stuff, although I did find [this](https://medium.com/@preitsma/setting-up-a-parity-ethereum-node-in-docker-and-connect-safely-f881faa17686) very helpful in getting started. Anyway, I hope this helps. 

## Disclaimer

All this is 100% new to me. At the time of first setting this up, I'd had the NAS box for less than one week, I had absolutely zero knowledge of Docker, I'd run a Parity node on my laptop, but had only the haziest idea of how it was working... so proceed with appropriate caution. There may well be better ways to do this: if you know of one, I'd be glad to hear about it.

## The Hardware

I'm running this on a [QNAP TS-253A](https://www.qnap.com/en-uk/product/model.php?II=211&ref=product_overview) NAS with 4GB memory. It has built-in support for Docker which is what makes all of this relatively straightforward, after getting one's head round Docker, that is.

As for networking, the kit is sitting at the end of a domestic home-plug network connected to the outside world by an approx. 18Mbps copper ADSL link. This is not high-end stuff. Even so, the node seems to be holding its own in comparison with others on ethstats.net.

# Set-up of the Parity node

## Initial set-up

### Via the GUI

Using the QNAP GUI in a Web browser I installed _Container Station_ which handles installing and monitoring Docker images and containers.

For my very first install of Parity I used the QNAP GUI app. In _Container Station_, select _Create Container_ and type "parity" in the search box. Available images will show up on the _Docker Hub_ tab.Next to "ethcore/parity" click "Create". It will ask you to select which version; select "latest". After acknowledging the pop-up disclaimer, you can then click "Create" on the _Create Container_ dialogue. Parity should now start running!

Back in the _Container Station_ Overview tab you can click on the running container and see what it's up to. Clicking on the arrows next to _Console_ will open up a new Web browser tab containing fuller output from the container. You should see Parity announcing its version, its ehash and then starting to sync.  My first sync was done overnight. The container hung up at some point near the end so I just restarted it and it was able to finish fine.

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

This requires using `ssh` to run a command line on the NAS.

First, use `docker cp` to copy the blockchain data you've already downloaded out of the container onto the NAS's own filesystem. I stopped the Parity container to do this, but I believe this may not be necessary.

```
docker cp my_parity_latest:/root/.parity /share/homes/admin/parity
```

This copies the whole of the container's _.parity_ directory, which includes the blockchain database directory, into a local directory on the NAS called _parity_. Mine is currently at 4.85GB. Not really a problem for a multi-terabyte NAS drive.

Note that _my\_parity\_latest_ is the name of my Parity container. Use your name here if you called it something different.

### Mounting the blockchain directory back on to containers

Now when we run a Parity image, we can attach the NAS's copy of the blockchain to the container and Parity will start syncing from there rather than right from the beginning. This is done with the `-v` flag to `docker run`:

```
docker run -d --name my_parity_latest -v /share/homes/admin/parity:/root/.parity etchcore/parity:latest
```

## Using a _docker-compose.yml_ file

It turns out that a convenient way of updating containers and storing configuration info is to use something called a [compose file](https://docs.docker.com/compose/overview/). These are called _docker-compose.yml_.  One of the advantages is that Docker Compose will automatically check for newer image files in the repository and download them accordingly.

Here's one that implements the configurations we've done so far.

```
ethereum-node:
  image: ethcore/parity:latest
  container_name: my_parity_latest
  volumes:
    - /share/homes/admin/parity:/root/.parity
```

Now all we need to do to update to the newest version of ethcore/parity:latest is the following (make sure you are in the same directory as the compose file). If there is no update it won't do anything - if you've edited the compose file, it will restart the container.

## Resource limits

You probably don't want Parity completely taking over your NAS box; it seems pretty happy to eat up however much memory you throw at it. You can limit the memory and CPU resources in the _Container Station_ GUI by clicking on the Parity container name (_my\_parity\_latest_ for me) and then on "Settings" in the top right. I've currently got the CPU set to 80% and memory limited to 2048MB. Until recently it ran fine with 1024MB, but has crashed a few times lately due to lack of memory, so I've thrown a bunch more at it. You don't have to restart the container (uncheck the box at the bottom), but it seems a good idea to do so if you are reducing the memory limit.

You can check the actual memory used by the container from the command line. Do a `cd` to `/sys/fs/cgroup/memory/docker/` and then `ls`. You will see container IDs listed - only one if you're just running Parity.

To see the memory actually in use by the container, substitute the container ID in the following.
```
cat CONTAINER_ID/memory.usage_in_bytes
```

To check that the memory limit has been applied, do this.
```
cat CONTAINER_ID/memory.limit_in_bytes
```

You can also try running `docker stats` on a container, though I've had odd results when running multiple containers. More info [here](https://www.datadoghq.com/blog/how-to-collect-docker-metrics/#toc-stats-command6).

Note that every time the container is updated the resource limits need to be reset manually. It's on my _Todo_ list to work out how to do this in the _docker-compode.yml_ file. 

# Linking with _ethstats.net_

## Introduction

Now you've got your Parity node running, you want it to show up on the very funky https://ethstats.net/ dashboard, right? Unfortunately, this is not entirely straightforward, but I'll tell you what's working for me.

Fortunately, a docker-based solution is available from the GitHub [eth-net-intelligence-api](https://github.com/cubedro/eth-net-intelligence-api) repository.

## Build the _ethnetintel_ docker image

Being a total Docker newbie, this initially had me scratching my head, but turned out to be pretty simple in the end.

First, download the [_Docker_ file](https://github.com/cubedro/eth-net-intelligence-api/blob/master/Dockerfile) from the repository.

Now, as per the instructions in the _Docker_ file, you ought to be able to type the following at the NAS command line.

```
docker build -t ethnetintel:latest .
```

This is supposed to build a docker image on the NAS, however it failed for me with an error. Instead, I did this step in a Linux VM I'd installed on the NAS, and then exported the image to the NAS filesystem using `docker save`. It's probably Docker version related. Nonetheless, it is a pain.

## Mounting app.json file and getting the password

There is a local configuration file required, _app.json_ which needs to be outside the container (similarly to the blockchain data above) so that it survives image updates and rebuilds. you can [download](https://github.com/cubedro/eth-net-intelligence-api/blob/master/app.json) this too from the repository. I put it in `/share/homes/admin/docker/ethnetintel/app.json` on the NAS filesystem.

The _app.json_ file is shared with/volume mounted on the container by adding the following lines to the _docker-compose.yml_ file for the _ethnetintel_ container, which I'm calling _my\_ethnetintel_.
```
  volumes:
    - /share/homes/admin/docker/ethnetintel/app.json:/home/ethnetintel/eth-net-intelligence-api/app.json
```

Some lines in this file need to be edited for your configuration. It's pretty self-explanatory.

One of the lines sets a WS_SECRET value. This you need to get hold of from someone in the know. I got it by asking nicely on the Parity [Gitter channel](https://gitter.im/ethcore/parity).

## Networking

The Docker file contains some instructions about networking that I didn't really understand at first, but after trying out a few different approaches it seems to be good advice.  Basically, set things up so that only the _my\_ethnetintel_ container has access to the outside world, and make the _my\_parity\_latest_ container share _my\_ethnetintel_'s network.

In the _docker-compose.yml_ file for _my\_parity_latest_ we strip out any "port:" lines that we added and replace them with this,
```
  net: 'container:my_ethnetintel'
```

This tells the Parity container to use _my\_ethnetintel_'s network.

## Modifications to docker-compose.yml

You can manage Parity and the Ethereum Net Intelligence tools with separate Docker compose files, or you can combine them (for which, see below). Running them separately means that when I restart the _my\_ethnetintel_ container, I also need to restart _my\_parity\_latest_ because it relies on the other container's networking. 

However, the Parity container can be stopped, started and updated while letting _my\_ethnetintel_ continue running.

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

The _my\_ethnetintel_ container seems fairly light-weight. I've currently allocated it 128MB (it's using about two-thirds of that now), and I've given it a 20% CPU cap. Both these settings are currently made in the QNAP _Container Station_ GUI and need to be redone every time the container is updated.

# Putting it all together

After rootling around in the Docker Compose documentation for a while, the following reflects my current workflow. 

1. Get the Parity image and extract the blockchain data to a filesystem outside the container.
2. Build the Ethereum Network Status image as described above.
3. Run the following _docker-compose.yml_ file with `docker-compose up -d`, after substituting your filesystem pathnames in the first part of each of the "volumes" sections.

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

In the couple of days since I've set this up, the _my\_ethnetintel_ container has twice stopped relaying block information to the Ethereum Network Status page, although Parity has continued running and relaying blocks quite happily. Stopping and restarting the containers (remember to start _my\_ethnetintel_ first!) gets things running again. For some reason, even after an outage of just a few seconds, it sometimes takes Parity several minutes to resync, during which it is hammering the disks constantly. I'm guessing that at startup it does some database reorganisation or something. Anyway, don't worry, it'll get there in the end.

I'm also recently getting frequent out of memory crashes, even when giving the containers a very big allocation - far more than they ought to need. Still incestigating this. First stop is to reboot the QNAP...

# Todo

* set resource limits (CPU/memory) in docker-compose.yml files
