---
layout:     post
title:      "Link Aggregations and Jumbo Frames in SmartOS"
date:       2017-11-09 00:25:23 -0500
categories: smartos systems
---
# Background

For a long while now I've been wanting to enable 802.3ad Link Aggregation (LACP) in conjunction with jumbo frames on my
[SmartOS][smartos] servers. This is a setup I'd used on Solaris 10, OpenSolaris, and Linux fileservers for years.
Unfortunately, due to SmartOS' unique installation and boot paradigm (where the OS is not installed locally, it is
extracted into a ramdisk from the boot archive at boot), getting it set up was not so easy. In fact, back when I set up
my first SmartOS server in 2013, [it wasn't possible][smartos-170].

That was rectified [and documented][smartos-manage-nics] (thanks [@rmustacc][github-rmustacc]!) a number of years ago,
but the one subsequent attmept to use it still ended in failure, so I gave up. Fast forward to yesterday, when I was
standing up my first new SmartOS server in a couple years and decided to give aggr+jumbos another shot.

[smartos]: https://www.joyent.com/smartos
[smartos-170]: https://github.com/joyent/smartos-live/issues/170
[smartos-manage-nics]: https://wiki.smartos.org/display/DOC/Managing+NICs
[github-rmustacc]: https://github.com/rmustacc

# Trying it out

The documentation isn't exactly clear on how to combine both options - aggregate interfaces are set up by hand in the
SmartOS `/usbkey/config` config file, but MTUs are set using `nictagadm(1M)`, which also appears to set up aggregate
interfaces. After looking through the contents of `nictagadm` (it's a shell script) and testing it out, it appears that
it's just a handy tool for setting the MTU on the correct entities in `/usbkey/config`.

With the config in what was hopefully the correct state, I rebooted and prayed. Unfortunately, the network
(`svc:/network/physical:default`) failed to start. Thankfully, SmartOS's custom SMF method for the network is very
chatty:

```sh-session
# cat /var/svc/log/network-physical:default.log
[ Nov  8 20:45:25 Executing start method ("/lib/svc/method/net-physical"). ]
[ Nov  8 20:45:25 Timeout override by svc.startd.  Using infinite timeout. ]
+ smf_configure_ip
+ /sbin/zonename -t
+ [ global = global -o shared = exclusive ]
+ return 0
+ LD_LIBRARY_PATH=/lib
+ export LD_LIBRARY_PATH
+ ADMIN_DHCP_TIMEOUT=300
+ ActiveAggrLinks=''
+ typeset -A ActiveAggrLinks
+ smf_netstrategy
+ smf_is_nonglobalzone
+ [ global != global ]
+ return 1
+ /sbin/netstrategy
+ set -- ufs none none
+ [ 0 -eq 0 ]
+ [ ufs = nfs ]
+ _INIT_NET_STRATEGY=none
+ export _INIT_NET_STRATEGY
+ typeset -A plumbedifs
+ smf_is_globalzone
+ [ global = global ]
+ return 0
+ /usr/sbin/dladm init-phys
+ log_if_state before
+ echo '== debug start: before =='
== debug start: before ==
+ /usr/sbin/dladm show-phys
LINK         MEDIA                STATE      SPEED  DUPLEX    DEVICE
bnx0         Ethernet             unknown    0      unknown   bnx0
bnx1         Ethernet             unknown    0      unknown   bnx1
+ /sbin/ifconfig -a
lo0: flags=2001000849<UP,LOOPBACK,RUNNING,MULTICAST,IPv4,VIRTUAL> mtu 8232 index 1
        inet 127.0.0.1 netmask ff000000 
lo0: flags=2002000849<UP,LOOPBACK,RUNNING,MULTICAST,IPv6,VIRTUAL> mtu 8252 index 1
        inet6 ::1/128 
+ echo '== debug end: before =='
== debug end: before ==
+ load_sdc_sysinfo
+ boot_file_config_enabled
+ load_sdc_config
+ load_sdc_bootparams
+ sed -e 's/,/ /g'
+ echo ''
+ create_aggrs
+ typeset links macs mode mtu
+ [[ -z aggr0 ]]
+ aggrs=( aggr0 )
+ eval links='${SYSINFO_Aggregation_aggr0_Interfaces}'
+ links=bnx0,bnx1
+ eval macs='${SYSINFO_Aggregation_aggr0_MACs}'
+ macs=78:2b:cb:1b:06:9d,78:2b:cb:1b:06:9e
+ eval mode='${SYSINFO_Aggregation_aggr0_LACP_mode}'
+ mode=passive
+ eval mtu='${CONFIG_aggr0_mtu}'
+ mtu=9000
+ [[ -z off ]]
+ echo 'Creating aggr: aggr0 (mode=passive, links=bnx0,bnx1)'
Creating aggr: aggr0 (mode=passive, links=bnx0,bnx1)
+ dladm create-aggr -l bnx0 -l bnx1 -L passive aggr0
+ [[ 0 -eq 0 ]]
+ add_active_aggr_links aggr0 78:2b:cb:1b:06:9d,78:2b:cb:1b:06:9e
+ typeset alink
+ ActiveAggrLinks[78:2b:cb:1b:06:9d]=aggr0
+ ActiveAggrLinks[78:2b:cb:1b:06:9e]=aggr0
+ [[ -n 9000 ]]
+ dladm set-linkprop -p mtu=9000 aggr0
dladm: warning: cannot set link property 'mtu' on 'aggr0': operation not supported
+ [[ 1 -ne 0 ]]
+ echo 'Failed to set mtu on aggr aggr0 to 9000'
Failed to set mtu on aggr aggr0 to 9000
+ exit 95
[ Nov  8 20:45:35 Method "start" exited with status 95. ]
```

What's going on here?:

```
dladm: warning: cannot set link property 'mtu' on 'aggr0': operation not supported
```

This was the very problem I'd had lo those many years ago, which, at the time, I'd attributed to a need to be able to
set the MTU on the individual interfaces before the aggegation, e.g.:

```sh-session
# dladm set-linkprop -p mtu=9000 bnx0
# dladm set-linkprop -p mtu=9000 bnx1
# dladm create-aggr -l bnx0 -l bnx1 -L passive aggr0
# dladm set-linkprop -p mtu=9000 aggr0
```

Well, my conclusion as to the source of the error was completely wrong. As it turns out, the reason `set-linkprop`
fails on the aggregation is because the Broadcom NetXtreme II `bnx` driver doesn't support runtime configuration of the
physical MTU limit. It can be set on IP interfaces, but not with `dladm set-linkprop`.

# Configuring bnx

The MTU must be configured through the `bnx` driver's config file, `bnx.conf`. The version that ships with the OS can
be found in `/kernel/drv/bnx.conf`. Some sources online and the [Solaris 11 docs][solaris-11-bnx] claim you need to set
the `Jumbo=...` option, but in the case of the driver version currently shipping in illumos, the option name is `mtu`.

But this is SmartOS, I can't just change `bnx.conf` and reboot like I would with traditional Solaris. Thankfully, I
stumbled across [a solution][nshalman-module-files] posted by [@nshalman][github-nshalman]. This clever enhancement by
[@wesolows][github-wesolows] and [@JohnSonnenschein][github-johnsonnenschein] allows arbitrary files to be added to the
boot archive ramdisk via the bootloader. As @nshalman says, @wesolows' [blog post][wesolows-anonymous-tracing] where
it's mentioned is pretty enlightening, as is the ["module" syntax documentation][multi-module-readme].

My hosts boot from PXE, and use iPXE as their bootloader. Thankfully, iPXE's [module (imgfetch)][ipxe-module] command
works in exactly the same way as the GRUB equivalent, and it pulls the file off my TFTP server just like the rest of
the boot components. Over on my (Debian) boot server, I now have:

```
# cat /var/lib/tftpboot/smartos/bnx.conf
mtu=9000,9000,9000,9000,9000,9000,9000,9000,9000,9000,9000,9000,9000,9000,9000,9000;
```

And my `menu.ipxe` entry for booting SmartOS has changed from:

```
kernel /smartos/${smartos-version}/platform/i86pc/kernel/amd64/unix -B smartos=true,console=${smartos-console},${smartos-console}-mode="115200,8,n,1,-" -v
initrd /smartos/${smartos-version}/platform/i86pc/amd64/boot_archive
boot
```
To:

```
kernel /smartos/${smartos-version}/platform/i86pc/kernel/amd64/unix -B smartos=true,console=${smartos-console},${smartos-console}-mode="115200,8,n,1,-" -v
module /smartos/${smartos-version}/platform/i86pc/amd64/boot_archive type=rootfs name=ramdisk
module /smartos/bnx.conf type=file name=kernel/drv/bnx.conf
boot
```

Regarding the `name` argument to the `type=rootfs` module, the [documentation][multi-module-readme] claims "`This
option is ignored for rootfs modules`". However, both @wesolows and @nshalman's examples use it, and I didn't test
without it, so I left it alone.

Reboot, and you should find that `bnx.conf` has been copied to `/system/boot/kernel/drv/bnx.conf`. At first I was
looking for it in `/kernel/drv/bnx.conf`, and thought it must not be working, but the
[documentation][multi-module-readme] clears this point up.

I also thought it wasn't working because even with `bnx.conf` in place and disabling all network configuration in
`/usbkey/config`, `dladm set-linkprop -p mtu=9000 bnx0` was still failing. It turns out that this is normal for the
`bnx` driver. The MTU cannot be set via `set-linkprop`, it must be done through `bnx.conf`. Thankfully, once you have
the aggregation configured correctly, the call to `dladm set-linkprop -p mtu=9000 aggr0` in
`/lib/svc/method/net-physical` does not fail.

Here is my working `/usbkey/config`:

```bash
#
# This file was auto-generated and must be source-able by bash.
#

aggr0_aggr=78:2b:cb:1b:06:9d,78:2b:cb:1b:6:9e
aggr0_lacp_mode=passive
aggr0_mtu=9000

admin_nic=aggr0
admin_mtu=9000
admin_ip=172.18.2.13
admin_netmask=255.255.255.0
admin_gateway=172.18.2.1

external_nic=aggr0
external_mtu=9000
external0_ip=128.118.250.13
external0_netmask=255.255.255.224
external0_gateway=128.118.250.1
external0_vlan_id=306

headnode_default_gateway=128.118.250.1

dns_resolvers=128.118.250.8,54.172.110.33
dns_domain=galaxyproject.org

ntp_hosts=0.smartos.pool.ntp.org
compute_node_ntp_hosts=dhcp

hostname=westmalle.galaxyproject.org
```

[solaris-11-bnx]: https://docs.oracle.com/cd/E86824_01/html/E54777/bnx-7d.html
[nshalman-module-files]: http://blog.shalman.org/overriding-driver-config-files-on-smartos/
[github-nshalman]: https://github.com/nshalman
[github-wesolows]: https://github.com/wesolows
[github-johnsonnenschein]: https://github.com/JohnSonnenschein
[wesolows-anonymous-tracing]: http://dtrace.org/blogs/wesolows/2013/12/28/anonymous-tracing-on-smartos/
[multi-module-readme]: https://github.com/wesolows/illumos-joyent/blob/6f4b830de6b4427d63fa12598c41301feebc4a42/README
[ipxe-module]: http://ipxe.org/cmd/imgfetch

# Conclusion

The main problem here seemed to be (other than my confusion at various points) the bnx driver. Many people report
having an overall better experience with Intel cards, which I do not doubt.

So, I'm happy to report that LACP + jumbo frames are working great (and probably have been for years), as long as you
can thwack your network device's driver in to submission.

Thanks as always to the Joyeurs for their hard work and support.
