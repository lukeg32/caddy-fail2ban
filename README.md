# caddy-fail2ban

A simple package to add [fail2ban](https://github.com/fail2ban/fail2ban) support to [caddy](https://caddyserver.com/). This simple module adds a `fail2ban` HTTP matcher based on a text file of IP addresses.
Forked from [Javex/caddy-fail2ban](https://github.com/Javex/caddy-fail2ban)

## Problem using fail2ban
when running a double proxy architecture fail2ban would either:  
* ban the upstream proxy, thus cutting every user off from the service (bad) (also just bad header assignment),  
* ban the client, which is meaningless as the client is connected through the proxy and never directly connects to the server (bad)  

fail2ban is blocking using firewall rules in iptables or nftables  

## Solution: 
Block connections based off of the X-Real-IP or X-Forwarded-For headers in the http (application) layer, but only when the remote is the trusted proxy.  
iptables/nftables arnt made to do this sort of behavior, but caddy should

## Reason for Fork
The orignal repo was supposed to fix this behavior but when I tried to use it, the client ip would get banned, as the client ip was getting passed back to the server properly (http headers).  
But when the client goes through the 2nd outer proxy, the original repo would compare the client ip to the proxies ip, thus back to the original problem.

## Changes:
1. if trusted proxy, grab ip from X-Real-IP or X-Forwarded-For and do the ban compare with that ip

Not a huge change, but now it works (also until I can figure out how to grab trusted proxy from caddy config, its just hard coded)

## Getting Started

First, make sure to build your caddy with support for this module:

```bash
 xcaddy build \
    --with github.com/lukeg32/caddy-fail2ban@main
```

Then insert this into your `Caddyfile`:

```Caddyfile
@banned {
	fail2ban ./banned-ips
}
handle @banned {
	abort
}
```

The right place for it depends on your setup, but you can find more complete examples in the [examples/](examples/) directory.

Next, you will need to create the fail2ban action. You can copy the suggested one if you like:

```bash
$ cp fail2ban/caddy-banfile.conf /etc/fail2ban/actions.d/caddy-banfile.conf
```

Now in any of your jails if you want to block requests at the HTTP layer, you can use the action:

```ini
action = caddy-banfile[banfile_path="/etc/caddy/banned-ips"]
```

The above path is the default so you can omit the `banfile_path` parameter if you like.

## Making Changes

If you would like to make your own changes, (ie set the trusted proxy to your own)
1. clone the repo some where
2. make your changes to the go files
3. build xcaddy with local repo

```bash
git clone https://github.com/lukeg32/caddy-fail2ban.git

# make your changes to the go files... nvim fail2ban.go

xcadddy build --with github.com/lukeg32/caddy-fail2ban@main=./caddy-fail2ban/
```

if all is successful it should build caddy with your changes

## Running tests

First run the go unit tests, then spin up a docker container to test the
integration with fail2ban

```
go build -v ./...
go test -v ./...

sudo docker build . -t caddy-fail2ban
sudo docker run --rm --name caddy-fail2ban --detach -v $PWD/test/Caddyfile:/etc/caddy/Caddyfile caddy-fail2ban
sudo docker exec -it caddy-fail2ban /usr/local/bin/caddy-fail2ban-test.sh
sudo docker logs caddy-fail2ban
sudo docker stop caddy-fail2ban
```
