# Overview

This is a guide to both the practical and theoretical concepts for creating a personal privacy server for proxying internet traffic. I wanted to understand the process better myself

What are we trying to do?
* Assume a different location
* Avoid profiling, tracking, surveillance
* Disassosiate home or business internet activity from ISP
* Strip ads, trackers, and junk to conserve bandwidth

How do we do this?
* Establish a secure communication channel with a trusted server
* Have that connection forward requests on our behalf
* Leverage that server to do additional processing

What are the benefits?
* Throw wrenches in data collection systems
* Create distance between yourself and your security work
* Avoid censorship and/or targeted surveillance
* Improve your browsing experience


## Theory

We need to create distance between our host and any other host we want to communicate with. So, what we need is a _proxy_. When it comes to proxies, they can be classed as *forward* or *reverse*. 
* Forward proxy (client request -> proxy -> server **response sent to client**)
* Reverse proxy (client request -> proxy -> server **response sent to proxy**)

In a gist: forward proxies (more or less) work on behalf of the client; reverse proxies (more or less) work on behalf of the server.

In this case, we need a forward proxy server which will:
* Accept client requests
* Process them according to our privacy constraints
* Pass them along on our behalf

We'll want to communicate with out proxy through a secure tunnel. Conveniently, we can use another proxy for that. What we'll have to do is point the traffic we want to send to our proxy through this tunnel while setting some reasonably secure practices for authenticating its setup.

Now that Windows comes natively with OpenSSH server, we can use that as a cross-platform, low permissions, and an easy way to establish a relatively secure tunnel. We'll need to provide SSH some instructions on what to do when it creates a tunnel to the proxy. We will ask it to accept connections to an arbitrary non-privleged port on the loopback interface. Traffic directed to 127.0.0.1:$PORT_NUMBER will be *dynamically forwarded* through the secure tunnel, to the proxy server, and then handled from there on.

ssh_config(5) explains:
```
DynamicForward
             Specifies that a TCP port on the local machine be forwarded over
             the secure channel, and the application protocol is then used to
             determine where to connect to from the remote machine.
             ...
             Currently the SOCKS4 and SOCKS5 protocols are supported, and
             ssh(1) will act as a SOCKS server.  Multiple forwardings may be
             specified, and additional forwardings can be given on the command
             line.  Only the superuser can forward privileged ports.
```
Cool, good to know. Key point of interest for us are the protocols, of which SOCKS4 and SOCKS5 are supported. The advantage of SOCKS5 for us in this case is that it supports DNS, so our name lookups can be proxied through the secure tunnel without an ISP having any knowledge. More on this in a minute.

So: we take a TCP port on localhost through the channel to a remote host (our privacy proxy) using SOCKS5 protocol. On the remote host, the application protocol is used to detemine next steps.

The schema will look like this: `socks5://127.0.0.1:$PORT_NUMBER`. Our goal then is to direct traffic from applications to this address on localhost, with the SSH SOCKS5 proxy server sending the traffic through the tunnel onwards towards the proxy server.

SOCKS5 is widely supported, for instance we can configure Firefox (better yet, a Firefox Profile) to send all traffic through this quickly. Or we can trust an extension like FoxyProxy to do it on demand or by URL pattern. `proxychains` will be used to demonstrate an on the fly way to proxy traffic through the tunnel, including whole `tmux` sessions.

## Procedure

### Acquire your server
  - Use your own discretion and trust with providers
  - Think traceability of your purchase if that risk factors for you.
### Select your OS
  - This guide assumes Ubuntu, mostly as its a learning exercise 
### Setup SSH communication
  - Note the IP of the server (we aren't setting up DNS records in this example)
  - Use the admin panel to upload your public key.
  - You'll probably login like this: `ssh -i ~/.ssh/id_ed25519 root@$SERVER_IP`. Verify you can.
  - Then, edit SSH config

```
SERVER_IP="YOUR IP ADDRESS"
SSH_KEY="YOUR SSH KEY"
ADMIN_USER="YOUR ADMIN USER"
PROXY_USER="YOUR PROXY USER"
SERVER_SSH_PORT=22
SOCKS5_PORT=42222

ssh-keygen -t ed25519 -b 4096 -C 'proxy' -f ~/.ssh/$SSH_KEY
# Get that key added to your server through the admin panel.
# Add Yukikey

ssh -i ~/.ssh/$SSH_KEY root@$SERVER_IP -vvv

echo '
Host privacy
  User $PROXY_USER
  Hostname $SERVER_IP
  IdentityFile ~/.ssh/$SSH_KEY
  IdentitiesOnly yes
  Port $SERVER_SSH_PORT
  DynamicForward 127.0.0.1:$SOCKS5_PORT
' >> ~/.ssh/config
```

<sup>1</sup> Requires additional scrutiny esp. when server in Five Eyes (or cooperating) nations.

### Configuring applications

#### Browser

For your favorite Firefox-based browser (I recommend LibreWolf)
- Navigate to: `about:profiles`
- "Create New Profile" and name it something relevant (i.e., privacy)
- Launch that profile
- Navigate to: `about:preferences` and type into search `proxy`
- `Configure Proxy Access -> Manual proxy -> SOCKS Host: 127.0.0.1 Port: $SOCKS5_PORT SOCKS v5 -> Proxy DNS when using SOCKS v5`

Or just dowload FoxyProxy and configure to needs

#### CLI

Many of your favorite tools may have native proxy support. For instance, cURL: `curl --proxy socks://localhost:$SOCKS_PORT ifconfig.me`

Some may not - notably recently, I noticed `amass` does not have native proxy support. Proxychains can be used with any TCP client application and can also chain proxys using proxychains such as if you wanted to make a portscan via proxy using `nmap` (but this has native proxy support). Amass, which does not, provides a good example: `proxychains amass enum -d evilcorp.com`. 

```
man proxychains(1)

FILES
       proxychains looks for config file in following order:

        ./proxychains.conf

        $(HOME)/.proxychains/proxychains.conf

        /etc/proxychains.conf
```

Let's create the `.conf` file in our `$HOME` folder. If we like it, we can apply system-wide `/etc/proxychains.conf`, or copy from it for our various projects `./proxychains.conf`

```
# OPTIONAL
# If you like all of your config in one place:
touch ~/.config/proxychains.conf
ln -s ~/.proxychains/proxychains.conf ~/.config/proxychains.conf

echo '
# proxychains.conf  VER 3.1
strict_chain
quiet_mode
proxy_dns
tcp_read_time_out 15000
tcp_connect_time_out 8000

[ProxyList]
socks5 127.0.0.1 $SOCKS5_PORT' > ~/.proxychains/proxychains.conf
```

On the client:


```
sudo adduser proxy
ssh-keygen -t ed25519 -b 4096 -C 'proxy' -f ~/.ssh/proxy_ed25519
scp ~/.ssh/proxy_ed25519.pub privacy:
```

Edit `~/.ssh/config` to use the new configuration
```

```
And test connection
```
ssh privacy
```
## Securing the host

### OS Security
```
sudo apt update && sudo dist-upgrade -y
sudo apt -y install zsh tmux dnsutils whois git gcc autoconf make lsof tcpdump htop tree apt-transport-https
```
### SSH Security

