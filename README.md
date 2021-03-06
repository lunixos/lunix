# lunix distribution

Declarative microservice-based linux distribution based on suckless principles
for desktop and embedded systems. In short, lunix

* uses the nix package manager,
* tries it's very hardest to push all user-space information through a pubsub
  (NATS) (sort of like ROS),
* is minimalistic and opinionated

What can you achieve with lunix?

* Like nixos, it's got "fearless upgrades" with rollbacks and all that jazzy
  package management magic
* Like ROS, it's got amazing extendability, but due to the protocol choice, is
  easier to use than the XMLRPC that ROS uses
* It's opinionated, but opinionated in a way that favors choice - most parts of
  lunix can be switched out
* The NATS pubsub interface allows for some very neat stuff, for example:


Sending a notice to toast notification when a USB stick has been plugged in

```
nats.subscribe('hardware.usb.add', (msg) => {
    nats.publish('toast.in', 'a USB stick was plugged in!');
});
```

Automounting said USB stick

```
nats.subscribe('hardware.usb.add', (msg) => {
    mount(msg.something, '/myplace');
});
nats.subscribe('hardware.usb.remove', (msg) => {
    umount('/myplace');
});
```

Creating a web page that displays your toasts over the interwebs:

```
nats.subscribe('toast.in', (msg) => {
    websocket.send(msg);
});
```

When your phone is connected to your device over bluetooth, open a
terminal on your desktop:

```
nats.subscribe('bluetooth.connected', (msg) => {
    nats.publish('wm.something', 'xterm');
});
```

When your IoT temperature sensor reports a low temperature reading,
and you're reading a book, spinlock to generate some heat

```
mqtt.subscribe('temperature0', (temp) => {
    if (temp <= 18) {
        nats.request('ps.calibre', (pid) => {
            while (1) {
                console.log('HEAT!');
            }
        }
    }
});
```

* All of these could be written in any programming language you like; node,
  rust, c, python.
* You can use node-red to configure your computer!


## Architecture

Lunix consists of

* A bootloader (choose your own; syslinux, grub, uboot, systemd-boot, ...).
  Just get a linux kernel going.
* A linux kernel.
* An init. You can spin your own here; systemd, runit, s6, openrc, etc, but
  lunix has it's own init you can use which gives you some further NATS
  integrations you can use - e.g. notifications when a service has started up
  and resource usage queries.
* A NATS-compatible pubsub server. Many parts of lunix is bound to the procotol
  specification of NATS, so this part is hard to switch out, but there will be
  NATS-compatible servers that are better suited for embedded use then NATS
  itself.
* The nix package manager. This is no requirement; it's very possible to run
  lunix using the alpine package manager, or any other package manager, but
  nix is a very neat package manager.
* 'natsdev' - a new hotplug daemon, like udev/eudev/mdev/smdev/..., but it's
  connection to NATS 
* A toast notification service
* A bluetooth notification service
* A network notification service,
* A NATS-connected PAM authentication module - sets readable and writeable
  topics for a user.
* An authentication service that `pam_nats` talks to
* and other useful stuff.


We also have some neat stuff to extend your setup, such as 

* An MQTT-NATS bridge,
* A websocket-nats bridge


## Design Specification

* Design with embedded in mind. All parts of lunix are designed to allow to be
  run in a low-cpu, low-memory, low-diskspace environment. 64 MB of RAM and 8
  MB NOR flash storage should be enough. Each package needs to be compatible
  with musl libc.
* No polling anywhere (in the base system), and no busy waiting/spinlocks. All
  lunix services are epoll based using proper linux fd-based primitives
  (sockets, timerfd, etc)
* Proper regular parsers everywhere - lots of extremely fast and secure ragel
  parsers.
* "Do one thing, and do it well"
* Polyglot environment - end-users can always integrate with lunix services 
  using whatever language and tool he/she is comfortable using
* Automatic upgrades in the background and other "automatics" is just annoying.
  Be very transparent.
* Don't try to do *everything* over NATS. We still use regular old 'getty',
  'login' and PAM. (Though we could provide a PAM module to give notifications
  that users have logged in). We still use X, optionally with a DM.
* Each part of lunix can be used on it's own, outside lunix.
* For really tight embedded environments, we're using buildroot instead of nix.
  We provide buildroot configuration for these packages.
* Be cloud-native. It should be easy to spin up lunix within a virtual machine,
  configure it to send syslogs somewhere useful, connect it to a larger NATS
  cluster, and push updates to it.


### lunix-init

The init system borrows ideas from existing init systems; runit, s6, systemd,
openrc and others. However, improves on them on some key points which merits
the existance of a new init.

It's able to watch a directory for changes like runit and s6 and automatically
starts/stops services that has been added/removed. This simplifies service
management into a simple "create service in the correct place". No need for
state resolving logic to figure out if a service needs to be restarted when
updated. It also integrates extremely well with the nix package manager,
because you can just create a new service directory, and every *changed
service* will be automatically started/restarted/stopped.

Runit and s6 writes to the service directories which makes it hard to keep them
in a read-only directory (like nix does). lunix-init does not.

lunix-init also provides service monitoring using NATS.


## Cloud-readyness

A design goal for lunix is to be cloud-ready. Thus, we want turn-key solutions
for creating a cluster of lunix machines which behave as a unified organism. To
do this, lunix provives easy (optional) mechanism to

* plug a lunix installation into an overlay network (slacks [nebula](https://github.com/slackhq/nebula)
  or wireguard)
* manage floating storage (something like ceph or something based on a private
  ipfs + brig or [dat](https://dat.foundation/)
* update machines remotely (using `nix copy`)
* plug in lunix into a NATS message cluster to manage
* easily spin up k8s on top of lunix

With this, it's easy to manage an extremely large fleet of machines, which can
be desktops, laptops, virtual machines, bare-metal servers, iot devices, and
everything else.


## Security

* Running lunix on an encrypted drive is easy
* Pre-configured stuff to provision PKI keys using HSMs
* Using Hashicorp Vault to provision keys should also be easy
* Using Yubikeys, PIV smartcards (any PKCS#11-capable device), GPG smartcards,
  or U2F keys to log into a lunix machine should be trivial. PIV support should
  be first-class so that companies managing lots and lots of machines can
  easily grant and revoke user access.


## Ideas

* It would be cool to try to integrate bedrock linux with lunix.
