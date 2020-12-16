# Running a VerusCoin Native Wallet (or Full Node) within TOR, properly

First things first, as of writing this (`2019-02-18`), there does not seem to be a way to use Agama via or within TOR properly if you expect privacy. 
Also, this guide only exists because it isn't just about adding `listenonion=1` to your config :-)

The Hidden Service part of this guide (`listenonion=1` in the Verus daemon config) may not be necessary, but it will benefit the general availability of VerusCoin within the TOR network at no privacy expense on the users end.

**Please, for your own safety, do not take this guide as the full and only truth. Do your own research. Think of this guide as a helper only. Explaining all possible TOR client setups clearly is beyond the scope of this guide. For example, DNS resolving is only roughly mentioned below!**

## Prerequisites

This guide assumes you're on a Debian-ish system. If you're not on Debian (or Devuan) itself, some details may be different, but the key concepts stay the same.

To start, we need to install `tor`. We'll also install `tor-arm` to get some overview.

```
sudo apt update
sudo apt -y upgrade
sudo apt -y install tor tor-arm
```

## Setup TOR

For further information about the TOR options mentioned in this paragraph, see:

```
man tor
```

Remember the last, bold-printed paragraph of the introduction? Good. 

Put this into your `/etc/tor/torrc`: 

```
DisableDebuggerAttachment 0
ClientOnly 1
SOCKSPort 9050
SOCKSPolicy accept 127.0.0.0/8
SOCKSPolicy reject *:*
ControlPort 9051
```

If any way possible, compile a list of 'trusted' TOR nodes. Use the `StrictNodes 1` and `EntryNodes [...]` options. Setup `DNSPort` to be `53` and overwrite your `/etc/resolv.conf` with `nameserver 127.0.0.1`. But be careful, screwing this up may lock you out of your server, if you're working on a remote machine. 
Change to your `/var/lib/tor` directory. We'll do 2 things to make `tor-arm` complain less.

```
cd /var/lib/tor
sudo -u debian-tor mkdir .arm
sudo -u debian-tor ln -s /etc/tor/torrc .arm/
```

`tor-arm` will give you quite a bit of insight into the state of your TOR client. To run it, do this:

```
sudo -u debian-tor arm
```

See it's manpage for details: 

```
man arm
```

Now, restart your TOR client.

```
sudo /etc/init.d/tor restart
```

## Setup the Verus wallet

To allow your wallet to setup a hidden service by itself, you have to add the user account from which the wallet is running to the `debian-tor` group like so: 

```
sudo gpasswd -a VERUSCOINUSERNAME debian-tor
```

These lines have to go into your `~/.komodo/VRSC/VRSC.conf`. Some may be there already, change them to below values.

```
listen=0
listenonion=1
onlynet=onion
bind=127.0.0.1:27485
proxy=127.0.0.1:9050
onion=127.0.0.1:9050
```

**Most important, remove any `bind=` statement that contains anything else than your loopback IP!!!**

Now restart your wallet.

`tail -f` on the `debug.log` file to make sure your wallet does connect somewhere and gets p2p updates. 

You should backup the `~/.komodo/VRSC/onion_private_key` file along with your `VRSC.conf` and `wallet.dat`, as it is (obviously) the private key to your onion hostname.

## Conclusion

**Again, i cannot stress this enough. DO NOT TRUST THIS GUIDE ALONE to establish your privacy regarding VerusCoin connections. Read about [TOR](https://www.torproject.org).** However, if you did follow the above steps correctly, all your VerusCoin related peer-to-peer traffic should now not only go through the tor network, but be properly untraceable as long as you are using zero knowledge-transactions correctly.

Probably, if privacy is a must-have for you, you may want to learn about [Tails OS](https://tails.boum.org/) if you haven't already.
