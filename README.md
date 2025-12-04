# cementbox: run untrusted code over SSH / in a disposable VM with mounts

## Why run stuff on other hosts?

My employer would like to see me using agentic AI.
On the other hand, I've got sensitive data
on both my main work system and on the company's network,
and of course I wanna isolate the AI tools from these two as much as possible.

So I run them in a disposable, easy-to-recreate libvirt/qemu VM
on a separate host that has no access to the company intranet.

## Features

YOLO mode with peace of mind:

* The best filesystem isolation I could come up with:
  running on a different host with just a few paths mounted over
  using a bubblewrapped sftp-server in reverse.
* The best network isolation I could come up with: running on another network.
* The best general isolation I could come up with: it's a VM on a separate host.
* The best cleanup I could come up with: reprovision the entire VM.

## Scripts

* `install-fedora`: a generic script to set up a disposable VM
  that can then be reprovisioned on a whim.
* `cementbox`: a generic script to run code on any remote host over SSH
  with parts of your filesystem mounted over, reverse-sshfs style.
* `example-cementboxed-install`: an example of using `install-fedora`
* `example-cementboxed-claude-setup`:
  an example of installing/configuring Claude Code inside the disposable VM.
* `example-cementboxed-claude`: an example of using `cementbox`
  to execute Claude Code inside the disposable VM.

Not sure how reusable they would be for your workflows as is,
but I hope they could serve as useful building blocks or
inspiration sources.

## I don't have a dedicated virtualization host, can I use just my only machine?

Yes, but you won't get that network isolation property I was after.
Still should be more secure and easy to clean up than bubblewrap/containers.
Use `qemu:///system` or `qemu:///user` for the `LIBVIRT_URL` then.

## Requirements

Virtualization host:
* `libvirt`
* `libvirt-ssh-proxy`
* `socat`
* `xmllint` (from libxml2)

Control host:
* `socat`
* `ncat` (from nmap-ncat)
* `bubblewrap` (for sftp-server)
* `/usr/libexec/openssh/sftp-server`
* `virsh` and `virt-install`
* shell/libvirt access to the virtualization host
* configuration for SSH vsock access (see below)

# Configuration for SSH vsock access

This lets you SSH into the VM without any bridging or port forwarding,
even when its network is down.
You'll need the following snippet in your ssh config (`~/.ssh/config`)
of your control host.
Substitute `$VM_HOSTNAME` and `$VIRT_HOST_HOSTNAME` for
the VM name and virtualization hostname/IP correspondingly.

```
Host $VM_HOSTNAME
      HostName qemu:system/$VM_HOSTNAME
      ProxyCommand ssh -T $VIRT_HOST_HOSTNAME 'cid=$(virsh -c qemu:///system dumpxml $VM_HOSTNAME | xmllint --xpath "string(//cid/@address)" -); exec ncat --vsock $cid 22'
      ProxyUseFdpass no
      CheckHostIP no
      UserKnownHostsFile ~/.ssh/$VM_HOSTNAME.known_host
```

## Potential follow-ups

* Implement and forward a proxy allowlisting access to the sensitive network.
