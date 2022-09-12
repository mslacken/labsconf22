---
title: Easy cluster deployment with warewulf v4
author: Christian Goll <cgoll@suse.com>
---
# Core concepts of warewulf v4
Warewulf's main purpose is to make HPC cluster deployment as easy as possible. An
HPC cluster normally contains a large number (up to several
thousands) of servers called compute nodes. In the ideal case all of these
nodes have the same physical components and are connected not only with a
management network, but also with a high speed network.

In order to ensure that all compute nodes use the same operating system, the 
operating system is booted over the management network and resides entirely in RAM which 
makes this installation method stateless.

Warewulf itself does not provide a method to create operating system images for the
compute nodes, but imports these images as OCI[^3] images. This makes it easy to import
any operating system image in a single step, as the specific node configuration is
stored in so-called overlays, which aim to be compatible with most Linux flavors.

# Operating system container
Operating system images can be imported to warewulf with the following methods

* as an OCI container archive as tar ball
* from an OCI container registry
* from a `chroot` directory on the filesystem of the warewulf host

At import time the operating system images are flattened to a simple `chroot` directory.
For convenience reasons a `resolv.conf` file for DNS resolution is copied over from the 
host to the container. Additionally the UIDs and GIDs can be synchronized.

For the deployment to the compute nodes compressed `cpio` images are created.
Warewulf also provides the ability to launch a shell within the container so that
additional packages can be installed in the container.

# Overlays
The configuration of the compute nodes and the relevant settings of the host system are 
managed by overlays. These overlays are Go templates[^1] and must have the _.ww_ suffix; see 
[issue.ww](#issue) for an example.
For the compute nodes following standard services are configured:

* network configuration
* SSH key for _root_
* NFS mounts
* the initial boot of the node

and for the warewulf host the following services are configured:

* exports
* dhpcd
* `/etc/hosts`

## Compute node overlays
Warewulf distinguishes between two kinds of overlays for the compute nodes.  The _system_
or _wwinit_ overlay is available at boot time, and does not get updated.
The so-called _runtime_ overlay becomes available after the network boot and is
updated with configurable time interval.

## Host overlays
The host overlays are only created if explicitly requested by the user, or if a 
referring configuration has been changed.

# Data model
All permanent configuration is stored in the file `node.conf` in the `yaml` format. This 
makes manual editing of this file possible, although the use of `wwctl` is encouraged.

Compute nodes can be configured explicitly or grouped together in so-called profiles. The
value of every configuration value can have three origins:

* a default value
* inherited from its profile
* explicitly set for the compute node

If an explicit value for a node is set it overrides the value
set from the profile, which itself takes precedence over the default value. The default value
can't be changed by the end user. 

As the origin of the values is tracked, it's easy to see how each value was set as one can see in 
the [output] of `wwctl node list -a`.

# Boot process
The whole design of warewulf is constructed around the boot process. This process uses iPXE [^2] which 
can create *one* big filesystem in RAM out of several cpio archives and boot a linux kernel on it. 
So the boot process has following steps:

1. BIOS pulls iPXE over PXE via TFTP
2. iPXE loads the kernel
3. iPXE creates magic initrd out of the container and wwinit cpio archives
4. iPXE boots kernel
6. kernel calls the `init` script provided in the `wwinit` overlay
7. `wwinit` configures ipmi
8. `wwinit` calls `init` of the magic initrd
9. system boots up
10. `wwclient`

# System services
## `warewulfd` system service
Warewulf provides its own daemon called `warewulfd` which serves the kernel, container and overlay
images via HTTP to the compute nodes.

Simple security is also provided, by allowing only downloads from a privileged port.

An individual asset tag can also be stored in the BIOS of every compute node to restrict the download 
to nodes which asset tag matches the one stored in warewulf.

## `wwclient`
The `wwclient` services is included in the _wwinit_ overlay and downloads the _runtime_ overlay on a 
regular interval.

# Examples
## Overlay template {#issue}
The _system_ overlay contains following file
```
Warewulf Node:      {{.Id}}
Container:          {{.Container}}
{{ if .Kernel.Version }}Kernel:             {{.Kernel.Version}} {{ end -}}
Kernelargs:         {{.Kernel.Args}}

Network:
{{- range $devname, $netdev := .NetDevs}}
    {{$devname}}: {{$netdev.Device}}
    {{$devname}}: {{$netdev.IpCIDR}}
{{if $netdev.Ipaddr6 }}    {{$devname}}: {{$netdev.Ipaddr6}}{{ end -}}
{{if $netdev.Hwaddr }}    {{$devname}}: {{$netdev.Hwaddr}}{{ end -}}
{{end}}
```
which will be rendered on e.g. _node01_ to
```
Warewulf Node:      node01
Container:          microOS2
Kernelargs:         quiet crashkernel=no vga=791

Network:
    default: eth0
    default: 10.10.1.11/24
    default: 52:54:00:08:a7:c0
```

## Output of `wwctl node list node01`
```
################################################################################
NODE                 FIELD              PROFILE      VALUE
node01               Id                 --           node01
node01               Comment            default      This profile is automatically included for each node
node01               Cluster            --           --
node01               Profiles           --           default
node01               Discoverable       --           true
node01               Container          default      tw
node01               KernelOverride     SUPERSEDED   leap15.3
node01               KernelArgs         --           (quiet crashkernel=no vga=791)
node01               SystemOverlay      --           (wwinit)
node01               RuntimeOverlay     --           (generic)
node01               Ipxe               --           (default)
node01               Init               --           (/sbin/init)
node01               Root               --           (initramfs)
node01               AssetKey           --           --
node01               IpmiIpaddr         --           10.10.10.10
node01               IpmiNetmask        default      255.255.255.0
node01               IpmiPort           --           --
node01               IpmiGateway        --           --
node01               IpmiUserName       --           admin
node01               IpmiInterface      --           --
node01               IpmiWrite          --           --
node01               default:DEVICE     --           eth0
node01               default:HWADDR     --           52:54:00:08:a7:c0
node01               default:IPADDR     --           10.10.18.11
node01               default:IPADDR6    --           --
node01               default:NETMASK    --           (255.255.255.0)
node01               default:GATEWAY    --           --
node01               default:TYPE       --           --
node01               default:ONBOOT     --           true
node01               default:DEFAULT    --           true
node01               default:TAG[mtu]   default      1500
```

[^3]: https://opencontainers.org/
[^1]: https://golangdocs.com/templates-in-golang
[^2]: https://github.com/ipxe/ipxe
