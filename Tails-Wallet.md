# Running a VerusCoin Wallet on Tails

This document describes a way to run a VerusCoin native wallet from within [Tails](https://tails.boum.org). Sadly, because Tails relies on TOR for anonymous networking, Agama isn't going to work, thus it will be a native commandline wallet.

**Please, for your own safety, do not take this guide as the full and only truth. Do your own research. Think of this guide as a helper only. Explaining all possible Tails options clearly is beyond the scope of this guide.**

## Prerequisites

Get a USB stick, at least 32GB in size, preferrably USB3. Follow [this guide](https://tails.boum.org/install/) to install the most recent release of Tails onto that USB stick.

To be able to complete this guide as well as anytime you want to use your VerusCoin wallet, you will have to set an [administration password](https://tails.boum.org/administration_password/) for Tails. Also, you need a `persistent volume` which is set to store at least `Personal Data` and `Dotfiles`.

## Build/obtain VerusCoin binaries

For now, `build.sh` sometimes fails via TOR. Appearantly, some download servers of opensource projects do not like TOR exit nodes. **What a shame.** However, that leaves us with 2 options:

1. Download and use premade binaries from [veruscoin.io](https://veruscoin.io)
2. Build the binaries on another system and copy them into your Tails instance.

For reference, here's a quick cheatsheet for building on Debian-ish systems:

```
sudo apt update
sudo apt -y install build-essential git pkg-config libc6-dev m4 g++-multilib autoconf \
	libtool ncurses-dev unzip git python python-zmq zlib1g-dev wget \
	libcurl4-openssl-dev bsdmainutils automake curl
git clone https://github.com/veruscoin/veruscoin
cd veruscoin
./zcutil/build.sh -j$(nproc)
strip src/verusd src/verus
```

Copy over these files:
```
veruscoin/src/verusd
veruscoin/src/verus
veruscoin/zcutil/fetch-params.sh
```

**NOTE: Should you choose option 1, use the preinstalled `GtkHash` tool to verify the checksums of your download.**

## Integrate wallet into Tails

Execute below steps in order to integrate the wallet into your Tails installation.

1. Create persistent `bin` dir and add it to path.

```
mkdir /live/persistence/TailsData_unlocked/dotfiles/bin
cp ~/.bashrc /live/persistence/TailsData_unlocked/dotfiles
```

Edit `/live/persistence/TailsData_unlocked/dotfiles/.bashrc` with your favourite text editor, put this at the end:

```
PATH=${PATH}:/home/amnesia/bin
export PATH
```

Copy over `verusd`, `verus` and `fetch-params.sh` to `/live/persistence/TailsData_unlocked/dotfiles/bin` and make sure all files are `chmod +x`.

2. Create custom `veruscoin-cli` and `veruscoind` scripts

These custom scripts are necessary because we have to set another data directory and need to fiddle with `iptables` before we can use the daemon. Feel free to name the scripts whatever you want, tho.

**`veruscoin-cli`**

Copy this into `/live/persistence/TailsData_unlocked/dotfiles/bin/veruscoin-cli`:
```
#!/bin/bash

${HOME}/bin/verus \
	-datadir=${HOME}/Persistent/VerusCoin \
	"$@"
```

Afterwards, `chmod +x /live/persistence/TailsData_unlocked/dotfiles/bin/veruscoin-cli` to make it executeable.

**`veruscoind`**

*When run, this will ask for your Tails administration password multiple times in order to check/set `iptables` rules. This is necessary in order to allow any communication from and to the wallet at all.*

Copy this into `/live/persistence/TailsData_unlocked/dotfiles/bin/veruscoind`:
```
#!/bin/bash -x

# save current working dir for later
OLDPWD="$(pwd)"

# determine configured rpcport (or use default value)
# then open port in firewall
RPCPORT=$(/bin/cat ${HOME}/Persistent/VerusCoin/VRSC.conf | /bin/grep rpcport | /usr/bin/cut -f2 -d=)
if [ -z "${RPCPORT}" ] || [ "${RPCPORT}" -neq "${RPCPORT}" ] > /dev/null 2>&1; then
	RPCPORT=27486
fi

# check if port is already opened, open if not
PORTCHECK=$(/usr/bin/sudo /sbin/iptables -L | /bin/grep ${RPCPORT})
if [ -z "${PORTCHECK}" ]; then
	/usr/bin/sudo /sbin/iptables -I OUTPUT -o lo -p tcp -s 127.0.0.1 -d 127.0.0.1 --dport ${RPCPORT} -j ACCEPT
fi

# change working dir to verus datadir
cd ${HOME}/Persistent/VerusCoin

# start veruscoin
${HOME}/bin/verusd \
	-datadir=${HOME}/Persistent/VerusCoin \
	-printtoconsole=1 \
	"${@}"

# return to old working dir
cd "${OLDPWD}"
```

Afterwards, `chmod +x /live/persistence/TailsData_unlocked/dotfiles/bin/veruscoind` to make it executable.

3. Download `zcash-params` and move to `dotfiles` directory

If you downloaded the binaries from [veruscoin.io](https://veruscoin.io), adapt below to `fetch-params` instead of `fetch-params.sh`.

```
cd ~
/live/persistence/TailsData_unlocked/dotfiles/bin/fetch-params.sh
mv ~/.zcash-params /live/persistence/TailsData_unlocked/dotfiles
```

4. Create and prepare data directory

**NOTE: Use the preinstalled `GtkHash` tool to verify checksums of the bootstrap download.**

```
mkdir -p ${HOME}/Persistent/VerusCoin/export; cd ${HOME}/Persistent/VerusCoin
wget http://bootstrapslc3ttl.onion/veruscoin/VRSC-bootstrap.tar.gz
wget http://bootstrapslc3ttl.onion/veruscoin/VRSC-bootstrap.tar.gz.md5sum
wget http://bootstrapslc3ttl.onion/veruscoin/VRSC-bootstrap.tar.gz.sha256sum
tar zxf VRSC-bootstrap.tar.gz
```

5. Create VRSC.conf

```
cat <<EOF >${HOME}/Persistent/VerusCoin/VRSC.conf
listen=0
listenonion=0
port=27485
proxy=127.0.0.1:9050
onlynet=onion

txindex=1

logtimestamps=1
logips=1
shrinkdebugfile=1

exportdir=${HOME}/Persistent/VerusCoin/export

server=1
rpcport=27486
rpcuser=veruscoin-${RANDOM}-$(whoami)
rpcpassword=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1)
rpcbind=127.0.0.1
rpcallowip=127.0.0.1
rpcthreads=1
rpcworkqueue=4

addnode=qxgvbauwyxshhp46.onion:27485
addnode=ndy4q5hqvgrq3moe.onion:27485
addnode=av3hnhrk5hhojvd2.onion:27485
addnode=qi65jg5qdfczziyl.onion:27485
EOF
```

6. Reboot Tails

This is necessary to get the changes to your `persistent volume` into the running system. Remember to set an [administration password](https://tails.boum.org/administration_password/) for Tails again after reboot.

7. Start VerusCoin

```
veruscoind -daemon 1>/dev/null 2>&1
tail -f ~/Persistent/VerusCoin/debug.log
```

*Congratulations, you have reached the end of this howto guide.*
