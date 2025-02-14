---
title: Bastille as FreeBSD Jail Management
published: 2024-01-17
description: "How I manage FreeBSD jails using Bastille."
tags: ["FreeBSD", "UNIX", "Operating System"]
image: "./cover.jpg"
category: Notes
draft: false
---

One of the features I like in FreeBSD is jail. Jail is a feature that allows us to create a virtual environment isolated from the main operating system. I use jail because I want to keep every installed application organized so things don’t get messy.

I preferred creating jails manually. You can check my configuration at [shinsha-jail](https://github.com/icaksh/shinsha-jail). Some might ask, why do it manually? I use vtnet as the jail network interface. I find it too complicated to configure vtnet manually when the jail is activated. Moreover, I need port forwarding for each jail I set up (perhaps similar to OpenVZ?). So, I prefer the manual approach rather than using existing tools—after all, they are just as complicated.

> Until, my shinsha-jail caused Google Cloud Engine (GCE) to be unable to boot when an additional bridge0 interface was added

I was actually quite skeptical about jail management tools because, the concept is the same, not much different from manual configuration. However, since I was tired of seeing my GCE serial console stuck at `DHCP renewal in 18000 seconds`, I decided to try using a jail management tool.

I tried ezjail, but... I didn’t really like it because I still had to configure pf manually. I have read `bastille`, reading `container` statement like `docker` make me sceptical. But, I decided to try it and I was surprised. Bastille is a great tool for managing jails. It is easy to use and has a lot of features.

## Installing Bastille

Bastille have a good documentation for someone who first using it. You can check it at [this post](https://bastillebsd.org/getting-started/).

## Why I Use Bastille

I use Bastille because it is easy to use and has a lot of features. I can create a jail with a single command. I can also easily manage jails, start, stop, and delete them. Bastille also has a template system that allows me to create jails with pre-configured settings.

What I love next that Bastille has a ZFS integration. I can create a jail with a ZFS dataset. This is very useful because I can easily manage the storage of the jail. I can also easily take snapshots of the jail and rollback to a previous state if something goes wrong. Bastille also has a function to easily mount a directory from the host to the jail. This is very useful because I can easily share files between the host and the jail.

I'am to lazy to configure `pf` manually. Bastille has a function to automatically configure `pf` for the jail. This is very useful because I can easily configure the firewall for the jail. I can also easily configure port forwarding for the jail.

So, let's try Bastille. I hope you find it useful. Feel free to reach out to me if you have any questions or feedback.

`// END`