# Iquidus Explorer for VerusCoin

## Server

A VPS with 4GB of RAM, anything above 20GB SSD storage and 2 CPU cores is the absolute minimum requirement. Start following the guide while logged in as `root`.


## Operating System

This guide is tailored to and tested on `Debian 9 "Stretch"`. Before starting, please install the latest updates:

```
apt update
apt -y upgrade
```

## Wallet

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

Now, lets create the data directory. Then, get the bootstrap and unpack it there.

```
mkdir -p ~/.komodo/VRSC
cd ~/.komodo/VRSC
wget https://bootstrap.0x03.services/veruscoin/VRSC-bootstrap.tar.gz
tar zxf VRSC-bootstrap.tar.gz
rm VRSC-bootstrap.tar.gz
```

Create `~/.komodo/VRSC/VRSC.conf` and include the parameters listed below, adapt the ones that need adaption.
A resonably secure `rpcpassword` can be generated using this command: 
`cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1`.

```
server=1
listen=1
listenonion=0
maxconnections=256

# wallet is not needed
disablewallet=1

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
bind=0.0.0.0
bind=::

# rpc settings
rpcuser=veruscoin
rpcpassword=your-secret-veruscoin-rpc-password
rpcport=27486
rpcthreads=256
rpcworkqueue=1024
rpcbind=127.0.0.1
rpcallowip=127.0.0.1

# if a peer jacks up more than 25 times in a row, ban it
banscore=25

# make sure gen is off
gen=0

# addnodes
seednode=185.25.48.236:27485
addnode=185.25.48.236:27487
seednode=185.64.105.111:27485
addnode=185.64.105.111:27487
seednode=185.25.48.72:27485
seednode=185.25.48.72:27487
```

For proper Iquidus operation, `txindex=1` is crucial. Afterwards, start the daemon and let it sync the small rest of the blockchain: 

```
komodod -ac_name=VRSC -ac_algo=verushash -ac_cc=1 -ac_veruspos=50 -ac_supply=0 -ac_eras=3 \
-ac_reward=0,38400000000,2400000000 -ac_halving=1,43200,1051920 -ac_decay=100000000,0,0 -ac_end=10080,226080,0 \
-ac_timelockgte=19200000000 -ac_timeunlockfrom=129600 -ac_timeunlockto=1180800 -addnode=185.25.48.236 \
-addnode=185.64.105.111 -daemon
```

To check the status and know when the initial sync has been completed, issue

```
komodo-cli -ac_name=VRSC getinfo
```

When it has synced up to height, the `blocks` and `longestchain` values will be at par. Additionally, you should verify against [the explorer](https://explorer.veruscoin.io) that you are in fact not on a fork. While we wait for this to happen, lets continue.

## NodeJS 10 & Prerequisites

Install NodeJS v10 like this: 

```
curl -sL https://deb.nodesource.com/setup_10.x | bash -
apt update; apt -y upgrade
apt -y install nodejs
npm -g install pm2
```
Alternatively, if you'd like to keep all the NodeJS-related data within a user account, you can use [nvm.sh](https://nvm.sh) to install NodeJS into the `veruscoin-explorer` account. See their notes for more info. 

## Iquidus Installation

Create a new user account for the explorer and switch to it.

```
useradd -m -d /home/veruscoin-explorer -s /bin/bash veruscoin-explorer
su - veruscoin-explorer
```

Now, check out the Veruscoin Iquidus repo and install it: 

```
git clone https://github.com/VerusCoin/explorer
cd explorer
npm install
cp settings.json.template settings.json
```

## Iquidus Configuration

See https://github.com/iquidus/explorer for the list of install instructions of Iquidus. The Genesis block/tx values are: 

```
//genesis
"genesis_block": "027e3758c3a65b12aa1046462b486d0a63bfa1beae327897f56c5cfb7daaae71",
"genesis_tx": "4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b",
```

You may want to disable the `twitter` and `markets` display. You should only bind Iquidus to loopback and proxy this to the world with nginx.


Make sure the VerusCoin wallet is running. You should now be able to start Iquidus.

```
cd /home/veruscoin-explorer/explorer; pm2 start --name explorer bin/instance
```

Display the logs with this command: 

```
pm2 log explorer
```

Initial sync will take up to 3 hours to complete. Start it like this:

```
cd /home/veruscoin-explorer/explorer && node scripts/sync.js index reindex
```

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

### Enable `logrotate` 

As `root` user, create a file called `/etc/logrotate.d/veruscoin` with these contents: 

```
/home/veruscoin/.komodo/VRSC/debug.log
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

### Simplify cli usage

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
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_tw_buckets = 2000000
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

The range is `1-100`. The *lower* the number, the *later* the system will start swapping stuff out. Run below command to activate the change, alternatively reboot the machine: 

```
sysctl -p /etc/sysctl.conf
```

### Install `molly-guard`

As a last sanity check before reboots, `molly-guard` will prompt you for the hostname of the system you're about to reboot. Install it like this: 

```
apt -y install molly-guard
```

Check `/etc/molly-guard/rc` for more options.
