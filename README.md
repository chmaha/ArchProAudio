# A Pro Audio Tuning Guide for Arch (and other Arch-based distros)
<p align="center">
  <img src="https://github.com/chmaha/ArchProAudio/assets/120390802/6095a4c2-0603-4c7e-98d7-eb81f3216332">
</p>

Following this guide will allow you to get the best possible performance on Linux for professional audio needs. Even though these steps are well-tested, it is wise to research what each step accomplishes and why. See also https://wiki.archlinux.org/title/Professional_audio.

_**Note for users of other distros**: Much of this guide can be adapted for other distros by simply switching out the package manager commands. For manually adding realtime privileges in other distros see [jackaudio.org](https://jackaudio.org/faq/linux_rt_config.html). In terms of kernels, Arch-based distros add the full preempt patch as part of the kernel config at build time whereas for others you might need to add `preempt=full` as a kernel parameter as part of step 4. From the yabridge documentation:_

> If the `uname -a` output contains `PREEMPT_DYNAMIC`, then run either `zgrep PREEMPT /proc/config.gz` or `grep PREEMPT "/boot/config-$(uname -r)"` depending on your distro. If `CONFIG_PREEMPT` is not set, then either add the `preempt=full` kernel parameter or better yet, switch to a kernel that's optimized for low latencies.

However, for Debian, Ubuntu or Arch, the low-latency [Liquorix kernel](https://liquorix.net/) might be an even better option. Install via:

```shell
curl -s 'https://liquorix.net/install-liquorix.sh' | sudo bash
```

With all other tweaks set identically, the Liquorix kernel performs noticeably better than the Debian kernel on my system. With Liquorix, there is no need to set `preempt=full` but the `threadirqs` parameter is still of great benefit.

---

## Quick-Start: Minimum Viable Setup

If you want to get up and running quickly, these are the highest-impact steps:

1. **Step 3** — Install realtime-privileges and add your user to the `realtime` and `audio` groups
2. **Step 4** — Add the `threadirqs` kernel parameter
3. **Step 5** — Set CPU governor to "performance" and disable sleep/screen locking

If you need to run Windows VST plugins, also follow **Step 11** (yabridge). If you are still experiencing audio performance issues after these steps, work through the full guide below.

---

## Pipewire

Pipewire is now the recommended audio server for pro audio on Arch. It handles ALSA, PulseAudio, and JACK compatibility in a single stack with lower latency overhead than the older JACK2 + PulseAudio combination.

Here's how to install pipewire if you are still running pulseaudio and jack:

```shell
yay -Rdd pulseaudio pulseaudio-alsa pulseaudio-jack jack2
yay -S pipewire-alsa pipewire-pulse pipewire-jack
```

Reboot then check whether pipewire and associated plugins are operational via:

```shell
inxi -Aa
```

For IRQ-based scheduling benefits when using ALSA, be sure to use the "Pro Audio" profile for your interface via your sound management tool.

**JACK vs. pipewire-jack:** For most DAW workflows, the native pipewire stack with the "Pro Audio" ALSA profile is the right choice. Use `pipewire-jack` (which provides a JACK compatibility layer) if you have software that explicitly requires a JACK server. You rarely need to run a separate JACK daemon anymore.

---

## Full In-depth Guide

### 1. Install Arch (or other favorite Arch-based distro)

_Optional:_
Install `yay`:

```shell
sudo pacman -S --needed git base-devel
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

In other Arch-derived distros, `yay` might already be installed or available via `sudo pacman -S yay`.

### 2. rtcqs (formerly known as realtimeconfigquickscan)

`rtcqs` audits your system configuration and reports which real-time audio settings are correctly configured. Run it both before and after applying the tweaks in this guide to verify improvements — green items indicate a passing check, red items indicate something that needs attention.

```shell
git clone https://codeberg.org/rtcqs/rtcqs.git
cd rtcqs
./src/rtcqs/rtcqs.py
```

### 3. Install realtime-privileges

```shell
yay -S realtime-privileges
```

Add your user to the `realtime` group (required for real-time scheduling) and the `audio` group (required to satisfy `rtcqs`):

```shell
sudo usermod -a -G realtime,audio $USER
```

Log out and back in, or reboot, for the group changes to take effect.

> **Note for non-Arch distros:** If `realtime-privileges` is not available in your package manager, you can manually configure real-time limits by editing `/etc/security/limits.conf` and adding:
> ```
> @audio   -  rtprio     95
> @audio   -  memlock    unlimited
> ```
> See [jackaudio.org](https://jackaudio.org/faq/linux_rt_config.html) for details.

### 4. Kernel tweak: threadirqs

The `threadirqs` parameter moves IRQ (interrupt request) handling off the main CPU thread and into dedicated kernel threads. This reduces latency jitter caused by hardware interrupts competing with your audio workload, and is one of the most effective single tweaks for pro audio performance.

**For GRUB:**

```shell
sudo nano /etc/default/grub
```

Change `GRUB_CMDLINE_LINUX=""` to `GRUB_CMDLINE_LINUX="threadirqs"`, then regenerate the GRUB config. On vanilla Arch:

```shell
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Some distros provide an `update-grub` wrapper that does the same thing.

**For systemd-boot:**

```shell
sudo nano /boot/loader/entries/arch.conf
```

Add `threadirqs` to the end of the `options` line, save, and reboot.

**For EndeavourOS and unified kernel image setups:**

```shell
sudo nano /etc/kernel/cmdline
```

Add `threadirqs` to the end of the options line, then run:

```shell
sudo reinstall-kernels
```

### 5. CPU Governor and Sleep/Screen Lock blocking

Set the CPU governor to "performance" to prevent the CPU from downclocking mid-session, which can cause audio dropouts. You can do this via your desktop environment. On KDE Plasma:

![image](https://github.com/user-attachments/assets/435a63c6-8b8a-4c20-a106-da249e702685)

Alternatively, set it via the terminal:

```shell
sudo cpupower frequency-set -g performance
```

Or add `cpufreq.default_governor=performance` as a kernel parameter (note that this can be overridden by power management tools like Powerdevil).

Also disable sleep and screen locking while recording or mixing to prevent interruptions.

> **Laptop users:** Running the performance governor continuously will increase heat and reduce battery life. Consider switching governors only during active recording/mixing sessions, or using a tool like `auto-cpufreq` to automate this.

### 6. Swappiness

Reducing swappiness tells the kernel to avoid swapping RAM to disk, which can cause unpredictable latency spikes.

```shell
sudo nano /etc/sysctl.d/99-sysctl.conf
```

Add:

```
vm.swappiness=10
```

### 7. Spectre/Meltdown Mitigations

If you run `rtcqs.py` and it gives you a warning about Spectre/Meltdown mitigations, you could add `mitigations=off` to `GRUB_CMDLINE_LINUX`.

**Warning:** Disabling these mitigations reduces your system's security against certain CPU-level exploits. Only consider this on a dedicated studio machine not used for general web browsing or untrusted code. See https://wiki.linuxaudio.org/wiki/system_configuration#disabling_spectre_and_meltdown_mitigations for more context.

### base-devel (as necessary)

```shell
yay -S base-devel
```

### 8. Install udev-rtirq

> **Pipewire users:** If you are using the "Pro Audio" ALSA profile in pipewire, IRQ thread priority is handled automatically and this step may not be necessary. If you are not using pipewire, or if `rtcqs` still reports IRQ priority issues, install udev-rtirq.

```shell
git clone https://github.com/jhernberg/udev-rtirq.git
cd udev-rtirq
sudo make install
reboot
```

### 9. DAW & Plugins

For regular work in a DAW, it is recommended to set its audio system to ALSA rather than going through pipewire or a JACK layer. Direct ALSA access bypasses the additional mixing layer, giving the lowest possible latency. If you need to monitor other audio sources during a session, in REAPER you can change the ALSA input and output devices to "default" (you need to type this manually):

![image](https://github.com/user-attachments/assets/956b4857-741e-4b5f-aba5-b5288e98bb37)

Set your desired audio device using your desktop environment.

**DAW Install Examples:**

REAPER:
http://reaper.fm/download.php or

```shell
yay -S reaper
```

Consider setting the RT (real-time) priority value to 80 on the audio device page. RT priority determines how aggressively the OS schedules the audio thread relative to other processes — higher values mean higher priority. A value of 80 matches the sane default used by Ardour and Mixbus, and is a reasonable starting point.

Ardour:
https://community.ardour.org/download or

```shell
yay -S ardour
```

Bitwig Studio:
https://www.bitwig.com/download/ (flatpak not compatible with yabridge) or

```shell
yay -S bitwig-studio
```

Also be sure to check out Qtractor, Tracktion Waveform, Mixbus, LMMS, Rosegarden, Zrythm etc.:
https://en.wikipedia.org/wiki/List_of_Linux_audio_software#Digital_audio_workstations_(DAWs)

#### Native plugins (`yay -S [pkgname]` where appropriate)

- My JSFX plugin collection (https://forum.cockos.com/showthread.php?t=275301)
- airwindows-git (http://www.airwindows.com/)
- x42-plugins (http://x42-plugins.com/x42/)
- lsp-plugins (https://lsp-plug.in/)
- zam-plugins (http://www.zamaudio.com/?p=976)
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

### 10. Wine-staging or Wine-tkg

> **Gentle Warning:** Recent WINE development has introduced compatibility issues affecting some Windows plugins on Linux — common symptoms include plugin crashes on load, GUI rendering failures, and MIDI timing problems. For the most current compatibility information, check the [yabridge "tested with" page](https://github.com/robbert-vdh/yabridge#tested-with) before installing a particular Wine version. A better long-term approach is to migrate entirely to Linux-native applications and plugins. The main gap currently is spectral editing — there are no Linux equivalents to iZotope RX or Acon Digital Acoustica.

Perhaps start with vanilla wine-staging and see how you fare. If your workflows rely heavily on sample-heavy VSTi like Kontakt, you may find better performance with wine-tkg (fsync enabled).

#### Wine-staging

```shell
yay -S wine-staging
```

Or install a specific version known to be compatible:

```shell
yay -S downgrade
sudo DOWNGRADE_FROM_ALA=1 downgrade wine-staging
```

Then pin the version to prevent it being upgraded:

```shell
# Add to /etc/pacman.conf:
IgnorePkg = wine-staging
```

#### Wine-tkg

Follow the instructions to git clone and install: https://github.com/Frogging-Family/wine-tkg-git/tree/master/wine-tkg-git#quick-how-to-

If using wine-tkg, set the `WINEFSYNC` environment variable to `1` according to https://github.com/robbert-vdh/yabridge#environment-configuration (depends on your display manager and login shell).

### 11. Install yabridge

```shell
yay -S yabridge yabridgectl
```

Or for the latest git version:

```shell
yay -S yabridge-git yabridgectl-git
```

Note: Depending on your distro, you might need to enable the multilib repo first:

```shell
sudo nano /etc/pacman.conf
```

Uncomment these lines:

```
[multilib]
Include = /etc/pacman.d/mirrorlist
```

Configure yabridge according to https://github.com/robbert-vdh/yabridge#readme, then install your Windows VST2, VST3, or CLAP plugins.

### 12. Check volume levels

Once everything is set up, verify that volume levels are set correctly. Run:

```shell
alsamixer
```

Check that output is set to 100 (vertical bars) or a gain of 0 dB (shown in the top left of alsamixer). Use F6 to select the correct sound card. You can also use your desktop environment's volume controls, though displayed numbers may not match alsamixer directly.

![alsamixer](https://user-images.githubusercontent.com/120390802/209148828-f5654838-eb25-4dd2-9955-4e0e8db99be2.png)

### 13. Diagnosing audio dropouts (xruns)

An xrun (overrun or underrun) occurs when the audio buffer is not filled in time, resulting in a click, pop, or dropout. To check whether you are experiencing xruns:

- **REAPER**: View > performance meter
- **Ardour/Mixbus**: The "Xrun" counter is displayed in the transport bar
- **pipewire**: Run `pw-top` in a terminal during a session to monitor real-time DSP load and missed deadlines
- **JACK (if used)**: `jack_iodelay` and `qjackctl`'s message window will report xruns

If you are consistently getting xruns, try increasing your buffer size first (e.g., 128 → 256 → 512 samples), then work back through this guide to identify the bottleneck.

### 14. Other useful tools (all available via the package manager)

**Music Player**: strawberry (can produce bit-perfect playback)  
![image](https://user-images.githubusercontent.com/120390802/209884991-d9901e4b-c242-4459-8127-060f2e86b9e1.png)  
**Tagger**: kid3, picard  
**DDP creation/verification/etc**: ddptools  
**Audio Converter**: fre:ac or soundconverter  
**CD Ripper**: asunder or cdrdao  
**CD Burner**: cdrdao, k3b or nerolinux4  
