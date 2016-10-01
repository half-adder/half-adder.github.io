---
layout: post
title: "Secret Management"
---

Managing secrets is a tricky problem. Managing them conveniently is an even tricker one. One of my first projects at my new job was to come up with a better system than the one that was in place. Here's how my team managed secrets when I started:

All secrets are stored in an encrypted file, let's call it ```enc-secrets.cfg```. The structure of this file is such that python's built-in [ConfigParser ](https://docs.python.org/2/library/configparser.html) will be able to read it.
