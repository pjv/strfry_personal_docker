# strfry Based Personal Nostr Relay Via Docker
This repo is basic configuration for running a personal strfry nostr relay (adding support for [NIP-20](https://github.com/nostr-protocol/nips/blob/master/20.md)] and [NIP-65](https://github.com/nostr-protocol/nips/blob/master/65.md)) via Docker.

This configuration is supplied in good faith as-is. This is not newbie-friendly; you need to know how to run public services on a Linux virtual private server (VPS) to be able to use this. Don't use this if you don't know how to use this (or if you do, please don't ask for help here on how to set up a linux server).

These configs assume a VPS host of either debian or ubuntu flavor. Other distros may need some tweaks.

## Requirements
You will need a plain VPS with Docker installed and a basic understanding of how to run services via docker compose.

If the VPS is behind any kind of firewall, you will need to configure it to accept incoming traffic on ports 80 and 443.

According to the strfry [deployment guide](https://github.com/hoytech/strfry/blob/master/docs/DEPLOYMENT.md), a relatively small 1 vCPU server with 2GB of RAM should be adequate for starting out. BTW, that guide recommends Vultr, but you can currently get a lot more server for the money with [Hetzner](https://www.hetzner.com/).

You will also need a domain name and the ability to create a DNS A (and optionally AAAA) record pointing to the IP address of your VPS.

## TL/DR Instructions
1. Choose a domain name for your relay and set up DNS A (and optionally AAAA) record(s) pointing from that name to the IPV4 (and IPV6) address(es) of your VPS.
2. From the command line on your VPS, clone strfry
`git clone https://github.com/hoytech/strfry.git`
3. Copy the following files and folders from this repo into your strfry clone (none of these filenames should collide with anything in the strfry repo):
* `my-strfry-db` (empty for now)
* `docker-compose.yaml`
* `my-strfry.conf`
* `write-policy.py`
* `Caddyfile`
4. Near the top of `write-policy.py` configure your whitelist by adding one or more hex pubkeys that are allowed to post events to your relay.
5. Starting at around line 30 in the file `my-strfry.conf` find the section called "info" and fill in the NIP-11 info for your relay. 
6. On the second line of `Caddyfile` replace the placeholder `your.relay.domain.name` with your relay's actual domain name.
7. Start your strfry relay by executing `docker compose up -d && docker compose logs -f`. If all goes well, you should see some logging from `strfry-caddy` about setting up an SSL certificate with Let's Encrypt for your relay's domain name. Once that's done your relay should be online and ready for your events.

## Caddy
This config uses [Caddy](https://caddyserver.com/) as a reverse proxy because it is extremely simple to configure and has built-in support for supplying and renewing SSL certs from Let's Encrypt. You can customize your Caddyfile a lot more than the barebones version in this repo (see the [docs](https://caddyserver.com/docs/)), but even this 6-line config is adequate to get your relay online with SSL.

## Write Policy Plugin
The file `write-policy.py` implements the following write policy for your personal relay:
1. Allow the pubkeys in your defined whitelist to post anything
2. Allows anyone to post kind 10002 events (See [NIP-65](https://github.com/nostr-protocol/nips/blob/master/65.md))
3. Allows anyone to post any event with a 'p' tag which includes any of the pubkeys in the whitelist (let's people write posts tagging you)
4. Rejects anything else



## Building Your Own

The primary `docker-compose.yml` file supplied here pulls a strfry docker image from a release on [my fork of strfry](https://github.com/pjv/strfry). If you inspect that repo you will see that I made two changes to the original strfry repo:

1. I added python to the docker image created in the original Dockerfile supplied with strfry (which lets the included write policy plugin run)
2. I created the github workflow that generates the docker image for the `docker-compose.yml` file mentioned above

You may prefer to build your own docker image instead of pulling mine. If so, you will need at least a 4GB VPS for the build process (I tried it on a 2GB VPS and the build did not complete - even with another 2GB of swap).

I have supplied an alternate docker-compose file called `docker-compose-build.yaml` which you can substitute in place of `docker-compose.yaml` in the instructions above along with two additional changes: 

1. You should also copy the file `my-Dockerfile` along with all the others in step 3
2. Before you start your strfry server the first time (step 7), you should execute `docker compose build` and then get a coffee because it will take a few minutes on a smaller VPS.

## Optional Fail2ban

If you run your server for a while and it becomes known in the nostr network, it will eventually start getting hammered by spammers trying to write to it. Your write policy plugin will not let them actually post, but the traffic can get overwhelming. 

So I've also included `strfry-fail2ban-filter.conf` and `strfry-fail2ban-jail.conf` that you can drop into `/etc/fail2ban/filter.d` and `/etc/fail2ban/jail.d` respectively to ban the IP addresses of abusers. The jail I've included bans for 2 hours at a time any host that the write policy plug blocks 10 or more times in a 2 minute window - you may want to adjust the parameters.

If you are familiar with docker, you may have noticed that the included compose file routes the container logs to the host's syslog. That is what lets fail2ban (running on the host) see the logs and block repeat offenders. It's also a good practice anyway with containers that log a lot to send their output somewhere that gets regularly rotated like a linux syslog does.

One more thing you need to do to enable fail2ban to work alongside docker. Docker will actually bypass the iptables block rules that fail2ban creates (letting the offending traffic through anyway). There may be [better ways](https://serverfault.com/questions/1043964/fail2ban-iptables-entries-to-reject-https-not-stopping-requests-to-docker-contai) to fix this, but on my server (since all it's doing is strfry), I've opted to just tell Docker not to create any iptables rules at all as mentioned [here](https://www.techrepublic.com/article/how-to-fix-the-docker-and-ufw-security-flaw/). The TL/DR is to add the following line to `/etc/default/docker`:

`DOCKER_OPTS="--iptables=false"`

...and then restart the `docker` service on your VPS.

Installing and configuring fail2ban is out of scope for this repo, but you're only one chatGPT prompt away from full instructions...

## Motivation
I think that in order for nostr to be what it can be, there needs to be a LOT of small relays all over everywhere as opposed to a short list of giant, "popular" relays that everyone uses which then must inevitably become centralized points of all manner of vulnerabilities that nostr is explicitly about avoiding.

Mike Dilger, author of the nostr client [gossip](https://github.com/mikedilger/gossip) as well as [NIP-65](https://github.com/nostr-protocol/nips/blob/master/65.md) wrote [The Gossip Model](https://mikedilger.com/gossip-model/) -- a description of how gossip manages relays in a way that can support this vision of having many smaller relays all over the place and still let people find the content they want to follow via nostr. In that document there is a section called **Personal Relays**. The _write policy_ in this repo is a first stab at implementing personal relays in that fashion.

## TODO
* Manage tag spam if / when strfry allows