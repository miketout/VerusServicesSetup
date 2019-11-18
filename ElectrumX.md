# ElectrumX for VerusCoin

## Server

A VPS with 4GB of RAM, anything above 40GB SSD storage and 2 CPU cores is the absolute minimum requirement. Start following the guide while logged in as `root`.


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
cp src/verusd src/verus ~/bin
strip ~/bin/verus*
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

For proper ElectrumX operation, `txindex=1` is crucial. Afterwards, start the daemon and let it sync the small rest of the blockchain:

```
verusd -addnode=185.25.48.236 -addnode=185.64.105.111 -daemon
```

To check the status and know when the initial sync has been completed, issue

```
verus getinfo
```

When it has synced up to height, the `blocks` and `longestchain` values will be at par. Additionally, you should verify against [the explorer](https://explorer.veruscoin.io) that you are in fact not on a fork. While we wait for this to happen, lets continue.

## Python 3.7 & Prerequisites

To install Python 3.7.1 from source, execute the below steps.

```
apt update; apt -y upgrade
apt -y install build-essential libssl-dev zlib1g-dev libncurses5-dev libncursesw5-dev libreadline-dev \
               libsqlite3-dev libgdbm-dev libdb5.3-dev libbz2-dev libexpat1-dev liblzma-dev tk-dev libffi-dev
cd /usr/src/
wget https://www.python.org/ftp/python/3.7.1/Python-3.7.1.tgz
tar zxvf Python-3.7.1.tgz
cd Python-3.7.1/
./configure --enable-optimizations
make -j$(nproc)
make altinstall
pip3.7 install multidict chardet
```

## ElectrumX Installation

Create a new system user for `electrumx`.

```
useradd -rMs /bin/false electrumx
```

Now, check out the Veruscoin ElectrumX repo and install it:

```
cd /usr/src
git clone https://github.com/VerusCoin/electrumx
cd electrumx; python3.7 setup.py install
```

## ElectrumX Configuration

Switch back to root. Copy over the `systemd` unit file and create `/etc/electrumx.conf`. Create a datadir and assign ownership to the `electrumx` user.

```
cp /usr/src/electrumx/contrib/systemd/electrumx.service /etc/systemd/system
cat <<EOF >/etc/electrumx.conf
COIN = Verus
DB_DIRECTORY = /electrumdb/VRSC
DAEMON_URL = http://veruscoin:your-secret-veruscoin-rpc-password@127.0.0.1:27486/
RPC_HOST = 127.0.0.1
RPC_PORT = 8000
HOST =
TCP_PORT = 10000
EVENT_LOOP_POLICY = uvloop
PEER_DISCOVERY = self
EOF
mkdir -p /electrumdb/VRSC && chown electrumx:electrumx /electrumdb/VRSC
```

Make sure the VerusCoin wallet is running. You should now be able to start ElectrumX.

```
systemctl start electrumx.service
```

Display the logs with this command:

```
journalctl -fu electrumx.service
```

Enable autostart with this command:

```
systemctl enable electrumx.service
```

Initial sync will take up to 3 hours to complete. Before that is done, ElectrumX will only allow RPC connections via loopback, but no external connections. To check ElectrumX status, do

```
electrumx_rpc getinfo
```

To see info about connected clients, execute

```
electrumx_rpc sessions
```


## Further considerations

None of the topics below is strictly necessary, but most of them are recommended.

### Improving SSH security
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
@reboot /home/veruscoin/bin/verusd -addnode=185.25.48.236 -addnode=185.64.105.111 -daemon 1>/dev/null 2>&1
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
