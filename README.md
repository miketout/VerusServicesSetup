# S-NOMP for VerusCoin Setup

## Server

A VPS with 4GB of RAM, anything above 20GB SSD storage and 2 CPU cores is the absolute minimum requirement. Start following the guide while logged in as `root`.


## Operating System

This guide tailored to and tested on `Debian 9 "Stretch"`. Before starting, please install the latest updates:

```
apt update
apt -y upgrade
```

## Poolwallet

The packages required in order to compile a VerusCoin wallet can be installed like this:

```
apt -y install build-essential git pkg-config libc6-dev m4 g++-multilib autoconf \
                   libtool ncurses-dev unzip git python python-zmq zlib1g-dev wget \
                   libcurl4-openssl-dev bsdmainutils automake curl
```


Create a useraccount for the wallet. Switch to that account.

```
useradd -m -d /home/veruscoin -s /bin/bash veruscoin
su - veruscoin
```

Now, clone the source tree and build the binaries:

```
git clone https://github.com/VerusCoin/VerusCoin
cd VerusCoin
./zcutil/fetch-params.sh
./zcutil/build.sh -j$(nproc)
```

After that is done, create a `~/bin` directory and copy over the binaries. Strip the debug symbols.

```
mkdir ~/bin
cp src/komodod src/komodo-cli src/komodo-tx ~/bin
strip ~/bin/komodo*
```

Start the VerusCoin daemon so we have a default configuration file:

```
komodod -ac_name=VRSC -ac_algo=verushash -ac_cc=1 -ac_veruspos=50 -ac_supply=0 -ac_eras=3 \
-ac_reward=0,38400000000,2400000000 -ac_halving=1,43200,1051920 -ac_decay=100000000,0,0 -ac_end=10080,226080,0 \
-ac_timelockgte=19200000000 -ac_timeunlockfrom=129600 -ac_timeunlockto=1180800 -addnode=185.25.48.236 \
-addnode=185.64.105.111 -daemon
```

Let it run for a few seconds and stop it again: 

```
komodo-cli -ac_name=VRSC stop
```

Edit the resulting `~/.komodo/VRSC/VRSC.conf` to include the parameters listed below, adapt the ones that need to be adapted.
A resonably secure `rpcpassword` can be generated using this command: 
`cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1`.

```
server=1
listen=1
listenonion=0
maxconnections=256

# logging related options
logtimestamps=1
logips=1
shrinkdebugfile=0

# how many blocks to check on startup
checkblocks=64

# indexing options
txindex=1
addressindex=1
timestampindex=1
spentindex=1

# make sure ipv4 & ipv6 is used
bind=<your public ipv4 address>
bind=<your public ipv6 address>

# rpc settings
rpcuser=veruscoin
rpcpassword=your-secret-veruscoin-rpc-password
rpcport=27486
rpcthreads=256
rpcworkqueue=1024
rpcbind=127.0.0.1
rpcallowip=127.0.0.1

# where to store exported data
exportdir=/home/veruscoin/export

# blocknotify
blocknotify=/usr/bin/node /home/s-nomp/s-nomp/scripts/cli.js blocknotify verus %s

# if a peer jacks up more than 25 times in a row, ban it
banscore=25

# stake if possible, although it's probably not helping much
gen=1
genproclimit=0

# addnodes
seednode=185.25.48.236:27485
addnode=185.25.48.236:27487
seednode=185.64.105.111:27485
addnode=185.64.105.111:27487
seednode=185.25.48.72:27485
seednode=185.25.48.72:27487
```

Afterwards, start the daemon again and let it sync the blockchain: 

```
komodod -ac_name=VRSC -ac_algo=verushash -ac_cc=1 -ac_veruspos=50 -ac_supply=0 -ac_eras=3 \
-ac_reward=0,38400000000,2400000000 -ac_halving=1,43200,1051920 -ac_decay=100000000,0,0 -ac_end=10080,226080,0 \
-ac_timelockgte=19200000000 -ac_timeunlockfrom=129600 -ac_timeunlockto=1180800 -addnode=185.25.48.236 \
-addnode=185.64.105.111 -daemon
```

Because this time `STDOUT` and `STDERR` are not redirected, it will produce a lot of output on the console, but will tell you at a glance wether or not the daemon has started syncing the blockchain successfully. Either open a new shell or exit the current one to regain silence. To check the status and know when the initial sync has been completed, issue

```
komodo-cli -ac_name=VRSC getinfo
```

When it has synced up to height, the `blocks` and `longestchain` values will be at par. Additionally, you should verify against [the explorer](https://explorer.veruscoin.io) that you are in fact not on a fork. While we wait for this to happen, lets continue.

## Redis 

Switch back to the `root` account by typing `exit` or hitting `CTRL-D`. Install Redis using `apt -y install redis-server`. In your `/etc/redis/redis.conf` file, make sure it contains this (and none of it is commented out):

```
bind 127.0.0.1
appendonly yes
```

Set `redis-server` to start at bootup and start it manually: 

```
update-rc.d redis-server enable
/etc/init.d/redis start
```

## Node.js

Still as `root`, install Node.js v8 like this: 

```
curl -sL https://deb.nodesource.com/setup_8.x | bash -
apt -y install nodejs

```

We will use [PM2](http://pm2.keymetrics.io) to manage NodeJS processes. Install it globally:

```
npm -g install pm2
```

## S-NOMP

S-NOMP and some of its dependencies will need additional packages in order to be built successfully: 

```
apt -y install libboost-all-dev libsodium-dev
```

Create a new user account to run S-NOMP from. Switch to that user and clone S-NOMP from miketouts repository: 

```
useradd -m -d /home/s-nomp -s /bin/bash s-nomp
su - s-nomp
git clone https://github.com/miketout/s-nomp
```

In `package.json`, change the `stratum-pool` dependency to `git+https://github.com/miketout/node-stratum-pool.git`. Next, install all dependencies using `npm`: 

```
npm update
npm install
```

Per default, S-NOMP will display `Sol` as the hashrate measuring unit. VerusCoin uses `H`, so lets change it:

```
cd ~/s-nomp
perl -p -i -e 's/Sol\/s/H\/s/g' libs/stats.js website/pages/stats.html website/static/stats.js website/static/miner_stats.js
```

In the web dashboard of the pool there is a 'Pool Luck' display which gives a rough estimate of how much time will pass between found blocks. To improve the accuracy of this number, change the value of `_blocktime` in line `584` of `libs/stats.js` to be closer to the actual block target of VerusCoin: 

```
cd ~/s-nomp
perl -p -i -e 's/_blocktime = 160/_blocktime = 55/g' libs/stats.js
```

Edit the `coins/vrsc.json` to look like below. **NOTE:** including the `burnFees` parameter is the crucial key part here.

```
{
    "name": "verus",
    "symbol": "vrsc",
    "algorithm": "verushash",
    "txfee": 0.0005,
    "requireShielding": true,
    "burnFees": true,

    "explorer": {
        "txURL": "https://explorer.veruscoin.io/tx/",
        "blockURL": "https://explorer.veruscoin.io/block/",
        "_comment_explorer": "This is the coin's explorer full base url for transaction and blocks i.e. (https://explorer.coin.com/tx/). The pool will automatically add the transaction id or block id at the end."
    }
}
```

Locate the `verushash` module directory. It either is `/home/s-nomp/node_modules/verushash` or `/home/s-nomp/node_modules/stratum_pool/node_modules/verushash`. In this directory, create a file called `index.json` containing this: 

```
module.exports = require('bindings')('verushash.node');
```

## Configuration Instructions

Shielding is required for mined VerusCoins. We will need 2 public and a z-address for this. Switch to the `veruscoin` user and generate the addresses:

```
komodo-cli -ac_name=VRSC getnewaddress
komodo-cli -ac_name=VRSC getnewaddress
komodo-cli -ac_name=VRSC z_getnewaddress
```

Next, we will dump the private keys of these addresses for safety reasons. For the public addresses, use 

```
komodo-cli -ac_name=VRSC dumpprivkey <public VerusCoin address>
```

For the z-address, use

```
komodo-cli -ac_name=VRSC z_exportkey <VerusCoin z-address>
```

**Save the data in an offline location, not on your computer!**

Now, switch to the `s-nomp` account. First, copy `~/s-nomp/config_example.json` to `~/s-nomp/config.json`. Edit it to reflect the changes listed below.

  * Under `clustering`, set `enabled` to `false`, **otherwise [PM2](http://pm2.keymetrics.io) fails to work.**
  * Set `stratumHost` to the external IP or DNS name of your server.

Note that [PM2](http://pm2.keymetrics.io) will take care of `clustering` by itself. Now create a pool config. Copy `~/s-nomp/pool_configs/examples/kmd.json` to `~/s-nomp/pool_configs/vrsc.json`. Edit it to reflect the changes listed below. 

 * Set `enabled` to `true`.
 * Set `coin` to `vrsc.json`.
 * Set `address` to one of the public addresses generated before.
 * Set `zAddress` to the z-address generated before.
 * Use the remaining public address for `tAddress`
 * Set `paymentInterval` to `180`
 * Set `minimumPayment` to `2`.
 * Set `maxBlocksPerPayment` to `8`.
 * There are 2 occurences of `user`, `password` and `port` each. Use the `rpcuser`, `rpcpassword` and `rpcport` values from `/home/veruscoin/.komodo/VRSC/VRSC.conf`.
 * Set `diff` to `131072`.
 * Set `minDiff` to `16384`.
 * Set `maxDiff` to `2147483648`

We are almost done now. Using the command mentioned at the beginning of this document, check if the blockchain has finished syncing. If not, wait for it to complete before continuing. 

Now switch to the `veruscoin` user, stop the wallet once more. 

```
komodo-cli -ac_name=VRSC stop
```

Edit `~/.komodo/VRSC/VRSC.conf` and add the blocknotify command below.

```
blocknotify=/usr/bin/node /home/s-nomp/s-nomp/scripts/cli.js blocknotify verus %s
```

Restart the wallet using the command already listed above. If you are not using `STDOUT`/`STDERR`-redirection, you will see errors about blocknotify. These are expected, because the pool is not running yet and thus the blocknotify script cannot complete successfully. 


Switch to the `s-nomp` user. Then start the pool using `pm2`: 

```
cd ~/s-nomp
pm2 start init.js --name s-nomp
```

Use `pm2 log` to check for S-NOMP startup errors. 

If you completed all steps correctly, the web dashboard on your pool can be reached via port `8080` on the external IP or the DNS name of your server.

## Further considerations

None of the topics below is strictly necessary, but most of them are recommended.

### Improving SSH security

If you remember the good old `rand=4; // chosen by fair dice roll` comic, you're probably doing this anyways. If you don't go google the comic, you might have missed a laugh there!

As `root`, generate a proper `/etc/ssh/moduli` like this:

```
ssh-keygen -G "/root/moduli.candidates" -b 4096
mv /etc/ssh/moduli /etc/ssh/moduli.old
ssh-keygen -T /etc/ssh/moduli -f "/root/module.candidates"
rm "/root/moduli.candidates"
```

Add the recommended changes from [CiperLi.st](https://cipherli.st) to `/etc/ssh/sshd_config`, also make sure that `PermitRootLogin` is at least set to `without-password`. Then remove and re-generate your host keys like this: 

```
cd /etc/ssh
rm ssh_host_*key*
ssh-keygen -t ed25519 -f ssh_host_ed25519_key < /dev/null
ssh-keygen -t rsa -b 4096 -f ssh_host_rsa_key < /dev/null
```

To finish, restart the ssh server: 

```
/etc/init.d/sshd restart
```

### Putting the pool behind some CDN

You should consider putting the webdashboard of your pool behind some CDN. A free CloudFlare account and any domain provider that allows changing the NS records of your domain will work. If you use a DNS name to point to your stratum ip, make sure to disable proxying for it!

### Reverse-proxying S-NOMP behind `nginx`

As `root`, install `nginx` and enable it on boot using these commands: 

```
apt -y install nginx
update-rc.d enable nginx
```

Create `/etc/nginx/blockuseragents.rules` with these contents: 

```
map $http_user_agent $blockedagent {
default         0;
~*malicious     1;
~*bot           1;
~*backdoor      1;
~*crawler       1;
~*bandit        1;
}
```

Edit `/etc/nginx/sites-available/default` to look like this: 

```
include /etc/nginx/blockuseragents.rules;
server {
	if ($blockedagent) {
		return 403;
	}
	if ($request_method !~ ^(GET|HEAD|POST)$) {
		return 444;
	}

	listen 80 default_server;
	listen [::]:80 default_server;
	charset utf-8;
	root /var/www/html;
	index index.html index.htm index.nginx-debian.html;

	location / {
		proxy_pass http://127.0.0.1:8080/;
		proxy_set_header X-Real-IP $remote_addr;
	}

	location /admin {
		rewrite ^/.* /stats permanent;
	}

}
```

Restart `nginx`: 

```
/etc/init.d/nginx restart
```

Switch to the `s-nomp` user, edit `/home/s-nomp/s-nomp/config.json` to bind the web interface to `127.0.0.1:8080`: 

```
[...]
"website": {
    "enabled": true,
    "host": "127.0.0.1",
    "port": 8080,
[...]

```

Restart the pool: 

```
pm2 restart s-nomp
```

If you've followed the above steps correctly, your pool's webdashboard is now proxied behind nginx.

### Disable unused webdashboard pages

Change to the `s-nomp` account. Edit `/home/s-nomp/libs/website.js` to have the `pageFiles` array look like below: 

```
var pageFiles = {
    'index.html': 'index',
    'home.html': '',
    'manual.html': 'manual',
    'stats.html': 'stats',
    'tbs.html': 'tbs',
    'workers.html': 'workers',
    'api.html': 'api',
    'miner_stats.html': 'miner_stats',
    'payments.html': 'payments'
}
```

### Link to the `payments` page

Change to the `s-nomp` user account. Edit `/home/s-nomp/website/index.html` to include a new link at the right position, which is somewhere in between lines `30-70`: 

```
<header>
[...]
    <li class="{{? it.selected === 'payments' }}pure-menu-selected{{?}}">
        <a class="hot-swapper" href="/payments">
            <i class="fa fa-usd"></i>&nbsp;
            Payments
        </a>
    </li>
[...]
</header>
```

### Enable `logrotate` 

As `root` user, create a file called `/etc/logrotate.d/pool` with these contents: 

```
/home/veruscoin/.komodo/VRSC/debug.log
/home/s-nomp/.pm2/logs/veruspool-out.log
/home/s-nomp/.pm2/logs/veruspool-error.log
{
  rotate 14
  daily
  compress
  delaycompress
  copytruncate
  missingok
  notifempty
}
```

### Autostart using `cron`

Switch to the `veruscoin` user. Edit the `crontab` using `crontab -e` and include the lines below:

```
@reboot /home/veruscoin/bin/komodod -ac_name=VRSC -ac_algo=verushash -ac_cc=1 -ac_veruspos=50 -ac_supply=0 -ac_eras=3 -ac_reward=0,38400000000,2400000000 -ac_halving=1,43200,1051920 -ac_decay=100000000,0,0 -ac_end=10080,226080,0 -ac_timelockgte=19200000000 -ac_timeunlockfrom=129600 -ac_timeunlockto=1180800 -addnode=185.25.48.236 -addnode=185.64.105.111 -daemon 1>/dev/null 2>&1
```

Switch to the `s-nomp` user. Edit the `crontab` using `crontab -e` and include the line below:

```
@reboot /bin/sleep 60 && cd /home/s-nomp/s-nomp && /usr/bin/pm2 start init.js --name s-nomp
```

### Simplify wallet usage

Switch to the `veruscoin` user. Create a file called `/home/veruscoin/bin/veruscoind` that looks like this: 

```
#!/bin/bash
OLDPWD="$(pwd)"
cd /home/veruscoin/.komodo/VRSC
/home/veruscoin/bin/komodod -ac_name=VRSC -ac_algo=verushash -ac_cc=1 -ac_veruspos=50 -ac_supply=0 -ac_eras=3 -ac_reward=0,38400000000,2400000000 -ac_halving=1,43200,1051920 -ac_decay=100000000,0,0 -ac_end=10080,226080,0 -ac_timelockgte=19200000000 -ac_timeunlockfrom=129600 -ac_timeunlockto=1180800 -addnode=185.25.48.236 -addnode=185.64.105.111 ${@}
cd "${OLDPWD}"
```

Create another file called `/home/veruscoin/bin/veruscoin-cli` that looks like this: 

```
#!/bin/bash
/home/veruscoin/bin/komodo-cli -ac_name=VRSC ${@}
```

Make both files executable: 

```
chmod +x /home/veruscoin/bin/veruscoin*
```

From now on, any time you would have to use the huge `komodod` or `komodo-cli` commands, you can just use them as shown below: 

```
veruscoind -daemon 1>/dev/null 2>&1
veruscoin-cli addnode 1.2.3.4 onetry
```

### Increase open files limit

Add this to your `/etc/security/limits.conf`: 

```
* soft nofile 1048576
* hard nofile 1048576
```

Reboot to activate the changes. Alternatively you can make sure all running processes are restarted from within a shell that has been launched _after_ the above changes were put in place.


### Networking optimizations

If your pool is expected to receive a lot of load, consider implementing below changes, all as `root`:

Enable the `tcp_bbr` kernel module: 

```
modprobe tcp_bbr
echo tcp_bbr >> /etc/modules
```

Edit your `/etc/sysctl.conf` to include below settings: 

```
net.ipv4.tcp_congestion_control=bbr
net.core.rmem_default = 1048576
net.core.wmem_default = 1048576
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.udp_rmem_min = 16384
net.ipv4.udp_wmem_min = 16384
net.core.netdev_max_backlog = 262144
net.ipv4.tcp_max_orphans = 262144
net.ipv4.tcp_max_syn_backlog = 262144
#net.ipv4.tcp_no_metrics_save = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_tw_buckets = 2000000
#net.ipv4.tcp_fin_timeout = 10
#net.ipv4.tcp_keepalive_time = 60
#net.ipv4.tcp_keepalive_intvl = 10
#net.ipv4.tcp_keepalive_probes = 3
#net.ipv4.tcp_synack_retries = 2
#net.ipv4.tcp_syn_retries = 2
net.ipv4.ip_local_port_range = 16001 65530
net.core.somaxconn = 20480
net.ipv4.tcp_low_latency = 1
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_mtu_probing = 1
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_limit_output_bytes = 131072
```

Run below command to activate the changes, alternatively reboot the machine: 


```
sysctl -p /etc/sysctl.conf
```

### Change swapping behaviour

If your system has a lot of RAM, you can change the swapping behaviour to only swap when necessary. Edit `/etc/sysctl.conf` to include this setting: 

```
vm.swappiness=1
```

The lower the number, the `later` the system will start swapping stuff out. Run below command to activate the change, alternatively reboot the machine: 

```
sysctl -p /etc/sysctl.conf
```

### Install `redis-commander`

As `root`, install `redis-commander` like this: 

```
npm -g install redis-commander
```

Consult `redis-commander --help` for more information.

### Install `molly-guard`

As a last sanity check before reboots, `molly-guard` will prompt you for the hostname of the system you're about to reboot. Install it like this: 

```
apt -y install molly-guard
```

Check `/etc/molly-guard/rc` for more options.
