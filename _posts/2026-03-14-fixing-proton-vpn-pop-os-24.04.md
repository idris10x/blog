---
layout: post
title: "Fixing Proton VPN Installation on Pop!_OS 24.04 LTS"
date: 2026-03-14
---


# Fixing Proton VPN Installation on Pop!_OS 24.04 LTS

## The Problem

I recently tried to install Proton VPN on my Pop!_OS 24.04 LTS system, but kept running into network errors. When attempting to download the Proton VPN repository package, I'd get:

```
Network is unreachable
```

Even more frustrating—other services like the npm registry worked fine, but anything related to apt repositories or certain domains would fail.

## Root Cause

After some troubleshooting, I discovered the issue: **my system had no IPv6 internet connectivity**, but my router's DNS server was returning IPv6 addresses for certain domains. When my system tried to connect to these IPv6 addresses, it failed silently (or appeared as "Network is unreachable").

## The Fix: Switch to Public DNS

The solution was to configure my system to use public DNS servers (Google DNS and Cloudflare DNS) that handle IPv4/IPv6 better:

```bash
sudo resolvectl dns wlp0s20f3 8.8.8.8 1.1.1.1
```

This immediately fixed the network issue. I could now resolve domains properly and download files.

## A New Challenge: Missing Sources File

With DNS working, I downloaded the Proton VPN repository package:

```bash
wget https://repo.protonvpn.com/debian/dists/stable/main/binary-all/protonvpn-stable-release_1.0.8_all.deb -O /tmp/protonvpn.deb
```

Then installed it:

```bash
sudo dpkg -i /tmp/protonvpn.deb
sudo apt update
```

But when I tried to install the GUI:

```bash
sudo apt install proton-vpn-gnome-desktop
```

I got:

```
E: Unable to locate package proton-vpn-gnome-desktop
```

Turns out, the Proton VPN package installed, but the repository sources file wasn't copied to the right location.

## The Final Fix

I manually extracted the sources file from the package and copied it to the correct location:

```bash
# Extract the package
dpkg-deb -x /tmp/protonvpn.deb /tmp/protonvpn-extract

# Copy the sources file to the right location
sudo cp /tmp/protonvpn-extract/etc/apt/sources.list.d/protonvpn-stable.sources /etc/apt/sources.list.d/

# Update and install
sudo apt update
sudo apt install proton-vpn-gnome-desktop
```
