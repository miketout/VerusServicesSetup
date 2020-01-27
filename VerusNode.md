# Verus Node with staking wallet

## Server

A VPS with 2GB of RAM, anything above 30GB SSD storage and 2 CPU cores that are able to handle AES-NI is the absolute minimum requirement. Start following the guide while logged in as `root`.


## Operating System

This guide tailored to and tested on `Debian 9 "Stretch"`. Before starting, please install the latest updates:

```
apt update
apt -y upgrade
```

## wallet

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
cp src/verusd src/verus src/verus-tx ~/bin
strip ~/bin/verus*
```

Now, lets create the data directory. Then, get the bootstrap and unpack it there.

```
mkdir -p ~/.komodo/VRSC
cd ~/.komodo/VRSC
wget https://bootstrap.veruscoin.io/VRSC-bootstrap.tar.gz
tar xf VRSC-bootstrap.tar.gz
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

# stake if possible, although it's probably not helping much
mint=1

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
verusd -daemon
```

To check the status and know when the initial sync has been completed, issue

```
verus getinfo
```

When it has synced up to height, the `blocks` and `longestchain` values will be at par. Additionally, you should verify against [the explorer](https://explorer.veruscoin.io) that you are in fact not on a fork. While we wait for this to happen, lets continue.

## Configuration Instructions

Shielding is required for mined / staked VerusCoins. We will need a public and a z-address for this. Switch to the `veruscoin` user and generate the addresses:

```
verus getnewaddress
verus z_getnewaddress
```

Next, we will dump the private keys of these addresses for safety reasons.
For the public address, use

```
verus dumpprivkey <public VerusCoin address>
```

For the z-address, use

```
verus z_exportkey <VerusCoin z-address>
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

### Autostart using `cron`

Switch to the `veruscoin` user. Edit the `crontab` using `crontab -e` and include the lines below:

```
@reboot /home/veruscoin/bin/verusd -daemon 1>/dev/null 2>&1
```

### Simplify wallet usage

Switch to the `veruscoin` user. Create a file called `/home/veruscoin/bin/veruscoind` that looks like this:

```
#!/bin/bash
OLDPWD="$(pwd)"
cd /home/veruscoin/.komodo/VRSC
/home/veruscoin/bin/verusd ${@}
cd "${OLDPWD}"
```

Create another file called `/home/veruscoin/bin/veruscoin-cli` that looks like this:

```
#!/bin/bash
/home/veruscoin/bin/verus ${@}
```

Make both files executable:

```
chmod +x /home/veruscoin/bin/veruscoin*
```

From now on, any time you would have to use the huge `komodod` or `komodo-cli` commands (both these commands are deprecated), you can just use them as shown below:

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
