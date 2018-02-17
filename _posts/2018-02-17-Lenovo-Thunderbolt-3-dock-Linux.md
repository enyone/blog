# Lenovo ThinkPad Thunderbolt 3 dock on Debian Linux

With these instructions you are able to make Lenovo ThinkPad Thunderbolt 3 dock work on Debian Linux still maintaining Thunderbolt Security settings turned on in bios.

end of TL;DR so now go and read...

The dock is [Lenovo 40AC0135EU](https://support.lenovo.com/fi/en/solutions/acc100356)

When surfing different web forums I found that Lenovo ThinkPad Thunderbolt 3 dock works well in Linux but that it is required to turn Thunderbolt security features off from bios. This is completely unacceptable, because Thunderbolt device can gain DMA access and thus dump the entire memory contents when attached to computer without you even noticing anything.

Do NOT turn off your Thunderbolt security features from bios ever.

As said the dock works without any effort with Linux kernel 4.14 and newer but with turning Thunderbolt security features off from bios. Reason is that even though Linux kernel is capable to intoduce Thunderbolt devices and (starting from 4.14) different Thunderbolt autorization modes for connected devices it is not offering any user friendly way to grant auhorized access to new connected Thunderbolt devices.

At the time of writing this article there was no distro providing any user firendly tools for end-user leaving communication with raw kernel device file handles the only possible thing left. Not quite as there is one fresh project evolving for publishing Thunderbold device autorization capabilities over D-Bus and it is [gicmo/bolt](https://github.com/gicmo/bolt)

You just clone it with git
```bash
$ git clone https://github.com/gicmo/bolt.git
```

Add required build dependencies
```bash
$ sudo apt install python3 python3-pip ninja-build \
  libpolkit-gobject-1-dev libumockdev-dev libudev-dev \
  libglib2.0-dev
$ pip3 install --user meson
```

And build it with the instructions at `gicmo/bolt README.md`
```bash
$ cd bolt/
$ ~/.local/bin/meson build
$ ninja -C build
$ ninja -C build test
```

And test the built binary works
```bash
$ ./build/boltctl --help
Usage:
  boltctl [OPTION...] [COMMAND]
```

And install it to your system
```bash
$ sudo ninja install
```

After rebooting you should be able to run boltctl command and see attached devices
```bash
$ boltctl 
 ● ThinkPad Thunderbolt 3 Dock
   ├─ uuid:        e0010000-0070-6f...
   ├─ vendor:      Lenovo
   └─ status:      connected
```

To authorize the dock run
```bash
$ boltctl authorize e0010000-0070-6f...
$ boltctl 
 ● ThinkPad Thunderbolt 3 Dock
   ├─ uuid:        e0010000-0070-6f...
   ├─ vendor:      Lenovo
   ├─ status:      authorized
   │  └─ security: secure
   └─ stored:      yes
      ├─ policy:   auto
      └─ key:      yes

```

Your dock now works including all usb3 ports, displayports, audio, and ethernet (which requires additional driver installation)

# Note for Debian 9.x users

Default location of D-Bus and Polkit configuration files when running `sudo ninja install` is `/usr/local/` and at least with my Debian it was unable to detect these files there. This can be fixed by manually moving following files
```bash
sudo mv /usr/local/share/dbus-1/interfaces/org.freedesktop.bolt.xml /usr/share/dbus-1/interfaces/
sudo mv /usr/local/share/dbus-1/system-services/org.freedesktop.bolt.service /usr/share/dbus-1/system-services/
sudo mv /usr/local/share/polkit-1/actions/org.freedesktop.bolt.policy /usr/share/polkit-1/actions/
sudo mv /usr/local/share/polkit-1/rules.d/org.freedesktop.bolt.rules /usr/share/polkit-1/rules.d/
sudo mv /usr/local/etc/dbus-1/system.d/org.freedesktop.bolt.conf /etc/dbus-1/system.d/
```
