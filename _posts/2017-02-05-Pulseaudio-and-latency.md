# Pulseaudio and latency

If you are Linux enthusiast and also are playing some instrument then you have probably encountered issues with Pulseaudio latency.

This blog post describes how to live with and manage Pulseaudio latency problems. For fast answer scroll directly to chapter [Positive effect on latency](#positive-effect-on-latency) as bellow is some leading information about Linux sound architecture

end of TL;DR so now go and read...

## Linux sound architecture

### Kernel level

Linux kernel has **snd** base module and on top of that other sound related modules can be built.

```ini
$ lsmod|grep ^snd|cut -d' ' -f1
snd_usb_audio
snd_usbmidi_lib
snd_rawmidi
snd_seq_device
snd_hda_codec_hdmi
snd_hda_codec_conexant
snd_hda_codec_generic
snd_hda_intel
snd_hda_controller
snd_hda_codec
snd_hwdep
snd_pcm
snd_timer
snd
```

Some of those modules creates log entries during activity or when loaded.

```ini
$ dmesg|grep 'snd\|card'|cut -d']' -f2
 snd_hda_intel 0000:00:1b.0: irq 45 for MSI/MSI-X
 input: HDA Digital PCBeep as /devices/pci0000:00/0000:00:1b.0/sound/card0/hdaudioC0D0/input12
 input: HDA Intel PCH Mic as /devices/pci0000:00/0000:00:1b.0/sound/card0/input13
 input: HDA Intel PCH Headphone as /devices/pci0000:00/0000:00:1b.0/sound/card0/input14
 input: HDA Intel PCH HDMI/DP,pcm=3 as /devices/pci0000:00/0000:00:1b.0/sound/card0/input15
 usbcore: registered new interface driver snd-usb-audio
```

### ALSA

[Advanced Linux Sound Architecture](http://www.alsa-project.org/)

**Note:** There is also other sound systems present in Linux world but ALSA is the most used currently.

Outputs are called as **playback** devices in ALSA.

```ini
$ aplay -l
**** List of PLAYBACK Hardware Devices ****
card 0: PCH [HDA Intel PCH], device 0: CX20590 Analog [CX20590 Analog]
	Subdevices: 1/1
	Subdevice #0: subdevice #0
card 0: PCH [HDA Intel PCH], device 3: HDMI 0 [HDMI 0]
	Subdevices: 1/1
	Subdevice #0: subdevice #0
card 1: CODEC [USB Audio CODEC], device 0: USB Audio [USB Audio]
	Subdevices: 1/1
	Subdevice #0: subdevice #0
card 2: Device [USB Sound Device], device 0: USB Audio [USB Audio]
	Subdevices: 0/1
	Subdevice #0: subdevice #0
```

Inputs are called as **capture** devices in ALSA.

```ini
$ arecord -l
**** List of CAPTURE Hardware Devices ****
card 0: PCH [HDA Intel PCH], device 0: CX20590 Analog [CX20590 Analog]
	Subdevices: 1/1
	Subdevice #0: subdevice #0
card 1: CODEC [USB Audio CODEC], device 0: USB Audio [USB Audio]
	Subdevices: 0/1
	Subdevice #0: subdevice #0
card 2: Device [USB Sound Device], device 0: USB Audio [USB Audio]
	Subdevices: 1/1
	Subdevice #0: subdevice #0
```

## Streams in general

Streams in computing are constructed using **sources**, **flows**, and **sinks**.

```ini
Source > Flow > Sink
```

For example audio data stream has a source (like sound card input) and sink (like sound card output) and between there is flow which modifies audio data. This modifying in flow can be as simple as gain control or delay control or more advanced like DSP effects.

## Pulseaudio

[Pulseaudio](https://www.freedesktop.org/wiki/Software/PulseAudio/) 

### Terminology

Pulseaudio is **stream** based. Both **sources** and **sinks** are present in terminology as well. Pulseaudio is also module based.

As Pulseaudio is not kernel based and does not introduce new kernel modules, some other way of managing audio data streams should be present. One example of these modules in this example setup is **module-alsa-card** which provides capability of interacting with ALSA hardware sound cards.

ALSA card's inputs are presented as **sources** in Pulseaudio.

```ini
$ pactl list sources short
0	alsa_input.usb-Burr-Brown_from_TI_USB_Audio_CODEC-00-CODEC.analog-stereo
		module-alsa-card.c	s16le 2ch 44100Hz	RUNNING
1	alsa_output.usb-0d8c_USB_Sound_Device-00-Device.iec958-stereo.monitor
		module-alsa-card.c	s16le 2ch 44100Hz	IDLE
2	alsa_output.pci-0000_00_1b.0.analog-stereo.monitor
		module-alsa-card.c	s16le 2ch 44100Hz	SUSPENDED
```

ALSA card's outputs are presented as **sinks** in Pulseaudio.

```ini
$ pactl list sinks short
0	alsa_output.usb-0d8c_USB_Sound_Device-00-Device.iec958-stereo	
		module-alsa-card.c	s16le 2ch 44100Hz	RUNNING
1	alsa_output.pci-0000_00_1b.0.analog-stereo
		module-alsa-card.c	s16le 2ch 44100Hz	SUSPENDED
```

### Latency and buffering

Every party described above adds own latency footprint into system. Linux kernel modules are generally well performing and kernel-level latency is often low enough. It is **buffering** between parties which causes major impact to latencies.

There is several ways to measure Pulseaudio latency. You cannot get rid of latency completely but sometimes default buffer sizes are too large and therefore cause worthless latency.

```ini
$ pactl list sinks
Sink #0
	State: RUNNING
	Name: alsa_output.usb-0d8c_USB_Sound_Device-00-Device.iec958-stereo
	Description: CM106 Like Sound Device Digital Stereo (IEC958)
	Driver: module-alsa-card.c
	Sample Specification: s16le 2ch 48000Hz
	Channel Map: front-left,front-right
	Owner Module: 7
	Mute: no
	Volume: front-left: 65536 / 100% / 0.00 dB,   front-right: 65536 / 100% / 0.00 dB
	        balance 0.00
	Base Volume: 65536 / 100% / 0.00 dB
	Monitor Source: alsa_output.usb-0d8c_USB_Sound_Device-00-Device.iec958-stereo.monitor
	Latency: 103758 usec, configured 100000 usec
	Flags: HARDWARE DECIBEL_VOLUME LATENCY SET_FORMATS 
```

Above sink details shows there is some latency present.

```ini
Latency: 103758 usec, configured 100000 usec
```

There is **current** latency and **configured** latency present in every source and sink. You cannot configure latency directly but you can modify variables that affect latency positively.

### Modules and scheduling

Several other Pulseaudio modules (other than just **module-alsa-card**) are present in this example setup.

```ini
$ pactl list modules short|cut -d$'\t' -f2
module-device-restore
module-stream-restore
module-card-restore
module-augment-properties
module-switch-on-port-available
module-udev-detect
module-alsa-card
module-native-protocol-unix
module-default-device-restore
module-rescue-streams
module-always-sink
module-intended-roles
module-suspend-on-idle
module-position-event-sounds
module-role-cork
module-filter-heuristics
module-filter-apply
```

Every one of these modules has it's own configuration parameters. Read more from [Pulseaudio modules documentation](https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/Modules/) 

Some of the parameters affecting positively to latency are:

* tsched
: * A boolean. If "true", the sinks and sources of this card will use the timer-based scheduling.

* fragments
: * The number of fragments to be used in the sink and source buffers. Only effective if the timer-based scheduling is disabled.

* fragment_size
: * The size of one fragment (in bytes) to be used in the sink and source buffers. Only effective if the timer-based scheduling is disabled.

* tsched_buffer_size
: * The total sink and source buffer size in bytes. Only effective if the timer-based scheduling is enabled.

* tsched_buffer_watermark
: * The buffer fill level (in bytes) at which the sinks must refill the buffer. Only effective if the timer-based scheduling is enabled.

* fixed_latency_range
: * A boolean. Normally when there's an alsa underrun or overrun, and timer-based scheduling is used, the alsa sink or source will raise the minimum latency that applications can get to avoid further underruns or overruns. If this option is enabled, the minimum latency will stay constant even if underruns or overruns occur.

#### Positive effect on latency

There is module **module-udev-detect** which searches ALSA sound cards present in system and adds own instance of module-alsa-card for each of these cards. Unfortunately default parameter values are not so nice for low-latency setup.

For example one of my USB sound cards has.

```ini
$ pactl list modules
...
Module #7
	Name: module-alsa-card
	Argument:	device_id="2"
				name="usb-0d8c_USB_Sound_Device-00-Device"
				card_name="alsa_card.usb-0d8c_USB_Sound_Device-00-Device"
				namereg_fail=false
				tsched=yes
				fixed_latency_range=no
				ignore_dB=no
				deferred_volume=yes
				use_ucm=yes
				card_properties="module-udev-detect.discovered=1"
```

I would suggest for you to try.

```ini
tsched=no
fixed_latency_range=yes
fragments=1
fragment_size=15
```

First, unload the module corresponding your desired sound card, #7 in my setup.

```ini
$ pactl unload-module 7
```

Then load it again with new parameter values.

```ini
pactl load-module module-alsa-card \
 device_id="2" \
 name="usb-0d8c_USB_Sound_Device-00-Device" \
 card_name="alsa_card.usb-0d8c_USB_Sound_Device-00-Device" \
 namereg_fail=false \
 tsched=no \
 fixed_latency_range=yes \
 ignore_dB=no \
 deferred_volume=yes \
 use_ucm=yes \
 card_properties="module-udev-detect.discovered=1" \
 fragments=1 \
 fragment_size=15
```

My latency with default values.

```ini
Latency: 103444 usec, configured 100000 usec
```

..which is ~103ms

And after changes.

```ini
Latency: 19369 usec, configured 20000 usec
```

..which is ~ 19ms

#### Chopping audio

If you audio starts to chop or stutter then your frame size is too small to keep sound card buffer saturated enough. You can play with `fragments` and `fragment_size` parameter values to find the best for your setup. Smaller gives lower latency, higher gives better chance to have unchoppy audio playback.

### Audio loopbacks

One way to connect a source directly to a sink is using **module-loopback**.

```ini
$ pactl load-module module-loopback latency_msec=1
```

And for unloading previous module.

```ini
$ pactl unload-module module-loopback
```

The other (and often better what comes to overall latency) way is to use **pacat** command and piping.

```ini
pacat -r --latency-msec=1 -d SOURCE-NAME | pacat -p --latency-msec=1 -d SINK-NAME
```

Use complete source/sink names.

```ini
pacat -r \
  --latency-msec=1 \
  -d alsa_input.usb-Burr-Brown_from_TI_USB_Audio_CODEC-00-CODEC.analog-stereo \
| pacat -p \
    --latency-msec=1 \
    -d alsa_output.usb-0d8c_USB_Sound_Device-00-Device.iec958-stereo
```
