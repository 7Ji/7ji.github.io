---
layout: post
title:  "Why OpenWrt DDNS does not start on boot"
date:   2025-06-05 15:50:00 +0800
categories: networking
---

On OpenWrt, DDNS functionality is provided by the opt-in `ddns-scripts` package (and optionally `ddns-scripts-[provider]` packages), which provides both an `rc-init` script `/etc/init.d/ddns` and a `hotplug.d` hook `/etc/hotplug.d/iface/95-ddns` to start it automatically:
- The `rc-init` script, instead of starting a "daemon" and maintaining it like other scripts that uses `procd`, mainly just calls `/usr/lib/ddns/dynamic_dns_updater.sh`, which without explicit interface names after `start/stop/reload` just double forks the workers and quits. Note it has an empty `boot()` function which shadows `start()` on boot, i.e. `/usr/lib/ddns/dynamic_dns_updater.sh -- start` would not be run on boot.

    ```sh
    cat /etc/init.d/ddns
    ```
    ```sh
    #!/bin/sh /etc/rc.common
    START=95
    STOP=10

    boot() {
            return 0
    }

    reload() {
            /usr/lib/ddns/dynamic_dns_updater.sh -- reload
            return 0
    }

    restart() {
            /usr/lib/ddns/dynamic_dns_updater.sh -- stop
            sleep 1 # give time to shutdown
            /usr/lib/ddns/dynamic_dns_updater.sh -- start
    }

    start() {
            /usr/lib/ddns/dynamic_dns_updater.sh -- start
    }

    stop() {
            /usr/lib/ddns/dynamic_dns_updater.sh -- stop
            return 0
    }

    ```
- The `hotplug.d` hook starts instances for "interfaces" when they're brought up by `netifd` and triggers `hotplug` event (e.g. when you `ifup` manually, or `reconnect` an interface from LuCI, or they start up automatically on boot after `netifd` is up and running):
    ```sh
    cat /etc/hotplug.d/iface/95-ddns
    ```
    ```sh
    #!/bin/sh

    # there are other ACTIONs like ifupdate we don't need
    case "$ACTION" in
            ifup)                                   # OpenWrt is giving a network not phys. Interface
                    /etc/init.d/ddns enabled && /usr/lib/ddns/dynamic_dns_updater.sh -n "$INTERFACE" -- start
                    ;;
            ifdown)
                    /usr/lib/ddns/dynamic_dns_updater.sh -n "$INTERFACE" -- stop
                    ;;
    esac
    ```

Both the `rc-init` script and the `hotplug.d` maintain nothing: they just spawn workers for interfaces, either fork and run `/usr/lib/ddns/dynamic_dns_updater.sh -n "$INTERFACE" -- start` by itself, or from a convenient shortcut provided by `/usr/lib/ddns/dynamic_dns_updater.sh -- start` which iterates uci config `ddns` internally to do the work.

So on boot the intended logic that `dynamic_dns_updater` shall be spawned on interfaces is as follows:
- The early init stage
- The procd `exec`-ed by early init and becomes new PID 1
- The `ubusd` becomes ready
- The `/etc/init.d/network` starts, and spawns `netifd` in `procd`
- The `/etc/init.d/ddns` starts, and due to empty `boot()` it does nothing
- The wan interface becomes ready in `netifd`
- The `/etc/hotplug.d/iface/95-ddns` hook triggers on interface(s) that you have configured `ddns` on, and the corresponding worker(s) would be spawned.

Note that the `hotplug.d` hook uses the internal name used by `netifd`. That is, an "physical" "interface" might e.g. be called as `br-lan` in the scope of Linux, but would be called `lan` in the scope of `netifd`, `uci`, `LuCI`, etc and of course `hotplug.d`. E.g.

Now let's discuss about an "issue": many with a `PPPoE wan` might find a strange phenomenon: even though they have "enabled" the `ddns` service and configured it on `pppoe-wan` "interface", the ddns worker would not correctly start on boot on their `PPPoE wan` interface. The reason this issue happens is due to the combination of following factors:
- In OpenWrt, software-based "interface"s are named in the style of `[protocol]-[network]`, e.g. for PPPoE-based "wan" interface/network, the actual Linux interface name that's created would be `pppoe-wan`
  ```
  config interface 'wan'
          option device 'eth5'
          option proto 'pppoe'
          option username 'xxxxxxxx'
          option password 'yyyyyy'
          option keepalive '10 60'
          option ipv6 'auto
  ```
  ```
  > ip l | grep wan
  19: pppoe-wan: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1492 qdisc fq_codel state UNKNOWN mode DEFAULT group default qlen 3
  ```
- [In luci-app-ddns](https://github.com/openwrt/luci/blob/55c93e60b4e598e81eeb1774d5d83ac32b245016/applications/luci-app-ddns/htdocs/luci-static/resources/view/ddns/overview.js#L899), the "interface" attribute is derived from the current source network/interface, e.g. when you configure "network" "wan", this would be `wan`; when you configure "interface" "pppoe-wan", this would be `pppoe-wan`:
  ```lua
  o = s.taboption('advanced', form.DummyValue, '_interface',
						_("Event Network"),
						_("Network on which the ddns-updater scripts will be started"));
  o.depends("ip_source", "interface");
  o.depends("ip_source", "network");
  o.forcewrite = true;
  o.modalonly = true;
  o.cfgvalue = function(section_id) {
    return uci.get('ddns', section_id, 'interface') || _('This will be autoset to the selected interface');
  };
  o.write = function(section_id) {
    var opt = this.section.formvalue(section_id, 'ip_source');
    var val = this.section.formvalue(section_id, 'ip_'+opt);
    return uci.set('ddns', section_id, 'interface', val);
  };
  ```
- [In dynamic_dns_updater.sh](https://github.com/openwrt/packages/blob/08b4fcd5e6b2ec5853c7eedd548bff0d3f541fbe/net/ddns-scripts/files/usr/lib/ddns/dynamic_dns_updater.sh#L135) i.e. the actual updater worker, the uci attribute `interface` needs to be the OpenWrt/netifd internal name that's put on the "network" / Openwrt "interface", not the Linux "interface":
  ```
  # interface 	network interface used by hotplug.d i.e. 'wan' or 'wan6'
  ```
- [In dynamic_dns_functions.sh](https://github.com/openwrt/packages/blob/08b4fcd5e6b2ec5853c7eedd548bff0d3f541fbe/net/ddns-scripts/files/usr/lib/ddns/dynamic_dns_functions.sh#L179), the `start_daemon_for_all_ddns_sections` takes the "network" name as argument and tries to get one `ddns` section with `interface` equalling it (note `wan` is the fallback name):
  ```sh
  # starts updater script for all given sections or only for the one given
  # $1 = interface (Optional: when given only scripts are started
  # configured for that interface)
  # used by /etc/hotplug.d/iface/95-ddns on IFUP
  # and by /etc/init.d/ddns start
  start_daemon_for_all_ddns_sections()
  {
      local event_if sections section_id configured_if
      event_if="$1"

      load_all_service_sections sections
      for section_id in $sections; do
        config_get configured_if "$section_id" interface "wan"
        [ -z "$event_if" ] || [ "$configured_if" = "$event_if" ] || continue
        /usr/lib/ddns/dynamic_dns_updater.sh -v "$VERBOSE" -S "$section_id" -- start &
      done
  }
  ```
- When retrieving network information from `netifd`, the "interface" must be the Openwrt "interface" / network, not the Linux "interface". There's no internal fallback logic to get the info from a Linux "interface".
  ```sh
  > ubus call network.interface status '{"interface":"wan"}' | jsonfilter -e '@["ipv4-address"][0].address'
  xxx.xxx.xxx.xxx
  > ubus call network.interface status '{"interface":"pppoe-wan"}' | jsonfilter -e '@["ipv4-address"][0].address'
  Command failed: Not found
  Failed to parse json data: unexpected end of data
  ```
- Likewise, the hotplug event only triggers on `wan`, not on `pppoe-wan`
- So, the `hotplug` event actually triggers and it runs `/usr/lib/ddns/dynamic_dns_updater.sh -n wan -- start` to start the worker for interface `wan`, but as it could not find any config section in `/etc/config/ddns` with `interface=wan` (which in reality is `interface=pppoe-wan`), it just quits and nevers spawns the actual `/usr/lib/ddns/dynamic_dns_updater.sh -S SECTION -- start` worker.

Note that while `hotplug.d` logic fails, you can still run `/etc/init.d/ddns start` to effectively run `/usr/lib/ddns/dynamic_dns_updater.sh -- start`, which just iterates the whole `/etc/config/ddns` config and would start all workers for all sections (as `-n NETWORK` is skipped and `-S SECTION` is run directly).

This of course does not only affect `PPPoE` `wan`, but in general affects any interface that's named differently from the corresponding network name.

There are two correct way to fix the issue, one is simply LuCI-only, and another one needs some uci (or manual config editting) but does not touch logic codes:
- The simple way is, without touching any of the above code, to configure your DDNS instance with source as "network" "wan", instead of "interface" "pppoe-wan", so you have `interface=wan` in your `/etc/config/ddns` and this way the `hotplug` event would correctly starts on "network" "wan"
- Another way is, to modify the `interface` value (you can also edit `/etc/config/ddns` manually)
  ```
  uci set ddns.cfv4.interface=wan
  uci commit ddns
  ```

Now with this knowledge you shall know that why the following "band-aid" "hacks" seem to "fix" the "problem" but they are very unreliable.
- By removing `boot()` function in `/etc/init.d/ddns`, you can force `/usr/lib/ddns/dynamic_dns_updater.sh -- start` to run on boot.
- By putting `/etc/init.d/ddns restart` in your `/etc/rc.local`, you're basically doing the same thing as removing `boot()`
- By putting both `/etc/init.d/ddns restart` in your `/etc/rc.local`, and a `sleep` before it, you have the addtional hope that `pppoe-wan` definitely becomes online after that timeout, however it's not guaranteed.
- By removing `boot()` function in `/etc/init.d/ddns`, putting `sleep` and `/etc/init.d/ddns restart` in your `/etc/rc.local`. You're combining "band-aid"s which makes your device more and more non-reproducible.
- Things call still fail after the above "band-aids" if your `pppoe-wan` connection is not there and you have configured `retry_max_count` for DDNS sections. If you use `hotplug.d` then the worker is guaranteed to started on `pppoe-wan` creation and stopped on `pppoe-wan` destruction.


