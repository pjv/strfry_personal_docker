# strfry Based Personal Nostr Relay Via Docker
This repo is basic configuration for running a personal strfry nostr relay supporting NIP-20 and NIP-65 via Docker.

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
* my-strfry-db (empty for now)
* my-Dockerfile
* docker-compose.yaml
* my-strfry.conf
* write-policy.py
* Caddyfile
4. Near the top of `write-policy.py` configure your whitelist by adding one or more hex pubkeys that are allowed to post events to your relay.
5. Starting at around line 30 in the file `my-strfry.conf` find the section called "info" and fill in the NIP-11 info for your relay. 
6. On the second line of `Caddyfile` replace the placeholder "your.relay.domain.name" with your relay's actual domain name.
7. Build your strfry image by executing `docker compose build`. This will take several minutes on a small VPS.
8. Start your strfry relay by executing `docker compose up -d && docker compose logs -f`. If all goes well, you should see some logging from `strfry-caddy` about setting up an SSL certificate with Let's Encrypt for your relay's domain name. Once that's done your relay should be online and ready for your events.

## Caddy
This config uses [Caddy](https://caddyserver.com/) as a reverse proxy because it is extremely simple to configure and has built-in support for supplying and renewing SSL certs from Let's Encrypt. You can customize your Caddyfile a lot more than the barebones version in this repo (see the [docs](https://caddyserver.com/docs/)), but even this 6-line config is adequate to get your relay online with SSL.

## Write Policy Plugin
The file `write-policy.py` implements the following write policy for your personal relay:
1. Allow the pubkeys in your defined whitelist to post anything
2. Allows anyone to post kind 10002 events (See [NIP-65](https://github.com/nostr-protocol/nips/blob/master/65.md))
3. Allows anyone to post any event with a 'p' tag which includes any of the pubkeys in the whitelist (let's people write posts tagging you)
4. Rejects anything else

This write policy plugin is written in python and requires one small change to the Dockerfile supplied in the strfry repo: the installation of the python language to the image running strfry. That is the only change in the file `my-Dockerfile` with respect to the Dockerfile supplied with strfry.

## Motivation
I think that in order for nostr to be what it can be, there needs to be a LOT of small relays all over everywhere as opposed to a short list of giant, "popular" relays that everyone uses which then must inevitably become centralized points of all manner of vulnerabilities that nostr is explicitly about avoiding.

Mike Dilger, author of the nostr client [gossip](https://github.com/mikedilger/gossip) as well as [NIP-65](https://github.com/nostr-protocol/nips/blob/master/65.md) wrote [The Gossip Model](https://mikedilger.com/gossip-model/) -- a description of how gossip manages relays in a way that can support this vision of having many smaller relays all over the place and still let people find the content they want to follow via nostr. In that document there is a section called *Personal Relays*. The _write policy_ in this repo is a first stab at implementing personal relays in that fashion.

## TODO
* Include fail2ban
* Manage tag spam if/when strfry allows