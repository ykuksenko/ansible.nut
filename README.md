Ansible Role: NUT (Network UPS Tools)
=========

Ansible role to configure NUT. This was designed to be pretty generic though it currently supports only the distros I use. Any contributions are welcome.

Requirements/Dependencies
------------

None.

Distribution support
--------------

 - Raspbian 8: netserver mode fully working
 - Fedora 24: netclient mode working (note name resolution issue at boot)
 - Arch: Researching possibilities as no native package seems to exist but my main server is arch...
 - Ubuntu: Untested (needs variable file)

Known Issues
-------
- upssched.conf, host.conf upsset.conf, upsstats.html are not implemented yet as I do not use it. Some require special handling as some require a specific rendering order. The dictionaries that are used above cannot be used here. This file needs to use arrays to preserve order.
- On Fedora 24 upsmon doesn't want to resolve DNS names for a NUT server at boot. It works fine if the service is bounced after boot. Alternatively using an IP just works. I am investigating this as I want DNS resolution.
- UDEV rules may not be present on the host with the UPS. If that is the case the role may fail when starting the USB drivers. After UDEV rules are installed you will need to unplug the UPS and replug it in. Then run the role again. This happens because something may bind to the USB device before the UDEV rules prevent this. (Educated guess)

Role Variables
--------------

- The vars folder contains examples of distribution specific variables.
- The defaults folder should contain examples of all available variables in each NUT config file. Note that these are rarely accessed by name from templates so if more are added in the future this role shouldn't need an update to use them. Just add additional parameters. 
- See the examples on how to use array variables.
- The defaults should always be overridden by the user.
- The default values reflect NUT defaults in Raspbian 8.
- You will find documentation for all config options at http://networkupstools.org/docs/man/.

Example Playbooks
----------------

Here are a couple working examples of how to use this role. Note that host names and IPs have been obfuscated.

Notes:

- {{nut_user}} comes from the distro specific variables in the var directory.
- Notice the LISTEN variable is an example of how to use the array type variables from the defaults/main.yml.
- Notice upsmon master entry is set to ''. This is interpreted differently by the template as this part of the config doesn't follow ini standards.
- {{ansible_default_ipv4.address}} comes from the setup module in ansible.

NUT Server that will monitor itself with a Cyberpower 1350C UPS:

    ---
    - hosts: ups
      become: yes
      vars:
      roles:
        - role: nut
          nut_nut_conf:
            MODE: netserver
            UPSD_OPTIONS: '""'
            UPSMON_OPTIONS: '""'
          nut_ups_conf:
            globals:
              maxretry: 3
              user: "{{nut_user}}"
            upses:
              ups01:
                driver: usbhid-ups
                port: auto
                desc: Primary UPS
                override.battery.charge.low: 20
                override.battery.charge.warning: 30
          nut_upsd_users:
            admin:
              password: password1
              actions: set
              actions: fsd
              instcmds: all
            monitor:
              password: password2
              upsmon master: '' #special entry, doesn't follow normal ini format.
          nut_upsd_conf:
            LISTEN:
              - "{{ansible_default_ipv4.address}} 3493"
              - 127.0.0.1 3493
          nut_upsmon_conf:
            DEADTIME: 15
            FINALDELAY: 60
            HOSTSYNC: 15
            MINSUPPLIES: 1
            MONITOR:
              - "ups01@localhost:3493 1 monitor password2 master"
            NOCOMMWARNTIME: 300
            POLLFREQ: 5
            POLLFREQALERT: 5
            POWERDOWNFLAG: /etc/killpower
            RBWARNTIME: 43200
            SHUTDOWNCMD: '"/sbin/shutdown -h +0"'
            RUN_AS_USER: "{{nut_user}}"

NUT Client that will monitor the above server.

    ---
    - hosts: client
      become: yes
      vars:
      roles:
        - role: nut
          nut_nut_conf:
            MODE: netclient
            UPSD_OPTIONS: '""'
            UPSMON_OPTIONS: '""'
          nut_upsmon_conf:
            DEADTIME: 15
            FINALDELAY: 5
            HOSTSYNC: 15
            MINSUPPLIES: 1
            MONITOR:
              - "ups01@10.10.10.243:3493 1 monitor password2 slave"
            NOCOMMWARNTIME: 300
            POLLFREQ: 5
            POLLFREQALERT: 5
            POWERDOWNFLAG: /etc/killpower
            RBWARNTIME: 43200
            SHUTDOWNCMD: '"/sbin/shutdown -h +0"'
            RUN_AS_USER: "{{nut_user}}"

License
-------

GPLv3

Author Information
------------------

Yevgeniy Kuksenko
