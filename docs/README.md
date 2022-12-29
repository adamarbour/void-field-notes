# Void Linux Install Notes

This site documents my experience installing [Void Linux]([Enter the void (voidlinux.org)](https://voidlinux.org/)) on my neat Lenovo ThinkPad X1 2nd Gen tablet. The device was purchased `R2/Ready for Reuse` as a tinkering device but has quickly become my favorite device in my arsenal.

The purpose of these notes is for me not to forget the steps taken to create what I would consider the perfect *nix setup and a guide for anyone else embarking on the same journey as me.

_Want to connect? Have questions? See something wrong? Find me @: 🌐 [https://github.com/adamarbour](https://github.com/adamarbour)_

**Why Void?**

This is an excellent question. Here is my reasoning:

- I installed Arch but was unhappy with systemd[^1]
- I installed Artix but it's just Arch with more work
- I was excited about Slackware finally updating (my original) but it still felt dated and I found myself ripping it apart to have what I wanted [^2]
- I've been digging the suckless philosophy[^3]
- I was intrigued with the Void distro pitch - _stable rolling, runnit, musl, and a package manager written from scratch... what is not to like?_

## Goals

I've broken this up into sub-guides that group my setup into logical parts that progressively enhance the experience and bring out the device's full potential.

- Goal #1: Install the base system and be able to boot into the void
- Goal #2: Improve the boot time and get all the hardware working
- Goal #3: Polish the terminal experience before I install a desktop experience
- Goal #5: Setup a desktop experience and eliminate the need for a mouse
- Goal #6: Optimize my workflow and make this my primary device

_NOTE: These goals are subject to change and may be ongoing The sub-sections will be updated as I make changes to my system. I'll put the revisions under a revision section should I make changes after working through the goal._

## Not Covered

These notes do not cover the following:

* How to create and install the bootable media
* How to prepare your HDD for encryption. I used the `Lenovo Drive Erase Utility` for my setup.
* How to dual-boot another distribution or Windows. This is a single distro device.

## Hardware

This device is pretty neat.  Some of the challenging (if not impossible) aspects of this device I want to take advantage of are:

1. The `productivity module` adds an extended battery and additional ports (included when I purchased the device).
2. The `onelink+` docking station.
3. The `presenter module`, which adds a detachable projector

I've laid out the technical specifications for the device below:

* __Processor:__ Intel® Core™ i7 7Y75 vProTM processor
* __Display:__ 12" 2K (2150 x 1440) IPS
* __Memory:__ 16 GB LPDDR3
* __Storage:__ 512 GB SSD OPAL2 PCIe-NVMe M.2
* __Graphics:__ Integrated Intel® HD Graphics 615
* __Camera:__ Front 2MP / Rear 8MP
* __Security:__ 
  * dTPM
  * Intel vPro
* __Audio:__ Realtek 2 stereo speakers & dual-array(noise-cancelling) microphones
* __Ports:__
  * 1 USB-C Power Delivery (PD)
  * 1 USB 3.0
  * Mini DisplayPort
  * microSD
  * NanoSIM
  * 3.5mm Headphone jack
* __Connectivity:__
  * Intel® Dual-Band Wireless-AC 8265 2 x 2 AC + Bluetooth® 4.2
* __Additions:__
  * _Stylus_ - ThinkPen Pro
  * _Productivity module_ - Adds an extended battery + 1x HDMI interface + 1x USB 3.0 + 1x Onelink+ port 
  * _Presenter module_ - Adds a modular projector attached to the LCD.

## References

[^1]: [No systemd](https://nosystemd.org/)
[^2]: [Oldest Active Linux Distro Slackware Finally Releases Version 15 (itsfoss.com)](https://news.itsfoss.com/slackware-15-release/)
[^3]: [Suckless Philosophy](https://suckless.org/philosophy/)



