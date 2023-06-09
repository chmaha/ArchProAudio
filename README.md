# A Pro Audio Tuning Guide for Arch (and other Arch-based distros)

Following this guide will allow you to get the best possible performance on Linux for professional audio needs. Even though these steps are well-tested, it is wise to research what each step accomplishes and why (the search engine is your friend :P ). See also https://wiki.archlinux.org/title/Professional_audio. 

For the archived Ubuntu/Debian-based guide see https://github.com/chmaha/UbuntuProAudio.

## Fundamentals

To get started after installing Arch, you could try just steps 3 and 5 below. If you need to use windows plugins on Linux also follow step 11 (easy: wine-staging, more advanced but potentially more performance: wine-tkg). Based on your individual pro audio needs, workflows, hardware specifications and more, your mileage may vary. If you are still having audio performance issues, try following the full guide...

### Pipewire?

Arch provides a relatively easy way of switching your audio to Pipewire (see https://pipewire.org/ and https://wiki.archlinux.org/title/PipeWire for more details). You may choose to wait until it ships as default in future releases although it is just as easy to roll things back. To switch to Pipewire run:
```shell
yay -S --needed pipewire-pulse pipewire-alsa pipewire-jack wireplumber
```
Be sure to say 'yes' to removing conflicting packages. Reboot! Check pipewire is operational via
```shell
inxi -Aa
```

It would also be wise to install a graph manager like qpwgraph to be able to make connections between apps and devices:
```shell
yay -S qpwgraph
```
That should give you everything you need to get up and running. I consider Pipewire ready for primetime at this point but for a dissenting view see https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=1000293#62.

#### Pipewire configuration

If you want to change the default samplerate, buffer size etc, you need to copy `/usr/share/pipewire/pipewire.conf` over to `/etc/pipewire/` and uncomment a few lines:

![2022-04-19_09-19](https://user-images.githubusercontent.com/90937680/163958025-f25a0f05-bc53-4fca-8f7f-28a94308407b.png)

To temporarily change samplerate/buffer size do not use PIPEWIRE_LATENCY environment variable. Instead, use:
```shell
pw-metadata -n settings 0 clock.force-rate <samplerate>
```
and
```shell
pw-metadata -n settings 0 clock.force-quantum <buffer-size>
```
To return to default values:
```shell
pw-metadata -n settings 0 clock.force-rate 0
```
and
```shell
pw-metadata -n settings 0 clock.force-quantum 0
```

See https://gitlab.freedesktop.org/pipewire/pipewire/-/wikis/Config-PipeWire#setting-buffer-size for more details.

#### Back to ALSA/Pulse/JACK ?

In the unlikely event you need to switch back:
```shell
yay -Rdd pipewire-alsa pipewire-pulse pipewire-jack
yay -S pulseaudio pulseaudio-alsa pulseaudio-jack jack2
```
Reboot then check whether pulseaudio is operational via
```shell
inxi -Aa
```
![inxi-pulseaudio](https://user-images.githubusercontent.com/120390802/230186674-26064d7e-314b-4bc9-b203-8792c951c458.png)
## Full In-depth Guide

### 1. Install Arch (or other favorite Arch-based distro)

_Optional:_
Install `yay`:

```shell
sudo pacman -S yay
```

Or, build from source:

```shell
sudo pacman -S --needed git base-devel
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

### 2. rtcqs (formerly known as realtimeconfigquickscan)

```shell
git clone https://codeberg.org/rtcqs/rtcqs.git
cd rtcqs
./src/rtcqs/rtcqs.py
```

### 3. Install realtime-privileges

```shell
yay -S realtime-privileges
```

Add user to "realtime" group and "audio" group (to satisfy `rtcqs`)

```shell
sudo usermod -a -G realtime,audio $USER
```

Log out/in or reboot...

### 4. Add "threadirqs" as kernel parameter

```shell
sudo nano /etc/default/grub
```

change 
`GRUB_CMDLINE_LINUX=""` to `GRUB_CMDLINE_LINUX="threadirqs"`

```shell
sudo update-grub
```

or if you don't have update-grub installed
```shell
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

### 5. Set governor to "performance"

i. Temporary:

```shell
sudo cpupower frequency-set -g performance
```

ii. Permanent:

Add `cpufreq.default_governor=performance` as a kernel parameter:

```shell
sudo nano /etc/default/grub
```

Line should now read: 

`GRUB_CMDLINE_LINUX="cpufreq.default_governor=performance threadirqs"`

```shell
sudo update-grub
```

or, for kernels < 5.9:

```shell
sudo nano /etc/default/cpupower # uncomment governor and change to performance
systemctl enable --now cpupower.service
systemctl start cpupower.service
```
    
### 6. Swappiness

```shell
sudo nano /etc/sysctl.d/99-sysctl.conf
```

add "vm.swappiness=10"
    
### 7. Spectre/Meltdown Mitigations

If you run `rtcqs.py` and it gives you a warning about Spectre/Meltdown Mitigations, you could add `mitigations=off` to GRUB_CMDLINE_LINUX. Warning: disabling these mitigations will make your machine less secure! https://wiki.linuxaudio.org/wiki/system_configuration#disabling_spectre_and_meltdown_mitigations

### base-devel (as necessary)

```shell
yay -S base-devel
```

### 8. Install udev-rtirq

```shell
git clone https://github.com/jhernberg/udev-rtirq.git
cd udev-rtirq
sudo make install
reboot
```

### 9. Jack2 + Jack D-Bus (__skip this step if you switched to Pipewire__)

```shell
yay -S qjackctl jack2-dbus
```

Enable Jack D-Bus interface:  
![image](https://user-images.githubusercontent.com/79659262/124497122-51218300-ddb2-11eb-8cb8-4bf873e026cd.png)


### 10. DAW & Plugins

Examples:

REAPER: 
http://reaper.fm/download.php or,  

```shell
yay -S reaper
```
Consider changing the RT priority value to 80 on audio device page. While RT priority numbers are all relative, this value matches the sane default used by Ardour and Mixbus.  

Ardour:
https://community.ardour.org/download or,

```shell
yay -S ardour
```

Bitwig Studio:
https://www.bitwig.com/download/ (flatpak not compatible with yabridge) or,

```shell
yay -S bitwig-studio
```

Also be sure to check out Qtractor, Tracktion Waveform, Mixbus, LMMS, Rosegarden, Zrythm etc...
https://en.wikipedia.org/wiki/List_of_Linux_audio_software#Digital_audio_workstations_(DAWs)

#### Native plugins (`yay -S [pkgname]` where appropriate)

- My JSFX plugin collection (https://forum.cockos.com/showthread.php?t=275301)
- airwindows-git (http://www.airwindows.com/)  
- lsp-plugins  (https://lsp-plug.in/)
- zam-plugins  (http://www.zamaudio.com/?p=976)
- distrho-ports (https://distrho.sourceforge.io/ports.php)
- dpf-plugins (https://distrho.sourceforge.io/plugins.php)
- elephantdsp-roomreverb (https://www.elephantdsp.com/)
- dragonfly-reverb (https://michaelwillis.github.io/dragonfly-reverb/)
- aether.lv2 (https://dougal-s.github.io/Aether/)
- Bertom Denoiser (https://www.bertomaudio.com/denoiser.html) (not in the Arch repos or AUR)
- sfizz / sfizz-git (https://sfz.tools/sfizz/)
- Chowdhury DSP (https://chowdsp.com/products.html) (available via AUR: `yay chow`)
- Auburn Sounds (https://www.auburnsounds.com/)
- TAL Software (https://tal-software.com/)
- Pianoteq (https://www.modartt.com/pianoteq)
- AudioThing (https://www.audiothing.net/)

### 11. Wine-staging or Wine-tkg

Perhaps start with vanilla wine-staging and see how you fare in terms of performance. If your workflows rely heavily on VSTi like Kontakt, you may find better performance with wine-tkg (fsync enabled). 

#### Wine-staging

```shell
yay -S wine-staging
```

Or, install a particular version that you know is compatible:

```shell
yay -S downgrade
sudo DOWNGRADE_FROM_ALA=1 downgrade wine-staging
```

Check https://github.com/robbert-vdh/yabridge#tested-with for up-to-date info.

OR...for the more adventurous:

#### Wine-tkg

Either download a wine-tkg build from https://github.com/Frogging-Family/wine-tkg-git/actions/workflows/wine-arch.yml or follow the instructions to git clone and install latest version: https://github.com/Frogging-Family/wine-tkg-git/tree/master/wine-tkg-git#quick-how-to-

If using wine-tkg, set the WINEFSYNC environment variable to 1 according to https://github.com/robbert-vdh/yabridge#environment-configuration (depends on your display manager and login shell)

### 12. Install yabridge

```shell
yay -S yabridge yabridgectl
```

or for latest git:

```shell
yay -S yabridge-git yabridgectl-git
```
    
Note: Depending on your distro, you might have to enable the multilib repo first:

```shell
sudo nano /etc/pacman.conf
```

and, uncomment these lines

```shell
[multilib]
Include = /etc/pacman.d/mirrorlist
```

Configure yabridge according to https://github.com/robbert-vdh/yabridge#readme  

then, install Windows VST2, VST3 or CLAP plugins!

### 13. Check volume levels!

Once everything is set up, don't forget to check that volume levels are set correctly. Whether using pipewire-alsa, pipewire-jack, vanilla ALSA or JACK, run
```
alsamixer
```
to check that output is set to 100 (vertical bars) or gain of 0dB (top left of alsamixer). Use F6 to select the correct soundcard. You can also use your desktop environment's volume controls if you have your interface enabled there but note that numbers don't seem to match alsamixer.

![alsamixer](https://user-images.githubusercontent.com/120390802/209148828-f5654838-eb25-4dd2-9955-4e0e8db99be2.png)

See section 5.1.5 of the Pipewire Arch documentation: https://wiki.archlinux.org/title/PipeWire

### 14. Other useful tools (all available via the package manager)

**Music Player**: strawberry (can produce bit-perfect playback)<br>
![image](https://user-images.githubusercontent.com/120390802/209884991-d9901e4b-c242-4459-8127-060f2e86b9e1.png) <br>
**Tagger**: kid3, picard <br>
**DDP creation/verification/etc**: ddptools <br>
**Audio Converter**: fre:ac or soundconverter <br>
**CD Ripper**: asunder or cdrdao <br>
**CD Burner**: cdrdao, k3b or nerolinux4


