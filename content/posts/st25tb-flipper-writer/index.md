---
title: "ST25TB Writer for Flipper Zero"
date: 2026-04-06
draft: false
tags: ["flipper-zero", "nfc", "st25tb"]
summary: "A Flipper Zero app for writing ST25TB NFC tags"
---

## Introduction

Recently I've acquired a Flipper Zero and started playing around with it to explore the world of NFC. I own some NCFs tags and started reading/writing them. I was able to do that with most of my tags, however, there was this specific tag, which was communicating with the ST25TB protocol, where I could only read from it. 

So I started looking online for some ways of writing on this tags using the Flipper Zero, however I didn't find anything for it.
In this post we will explore the steps I took to create an application that can write on these tags.

## How it works

The app opens a file browser, lets you pick an `.nfc` dump, and writes it block-by-block to an ST25TB tag placed on the Flipper.

## The code

```c
#include <furi.h>
#include <gui/gui.h>
#include <nfc/nfc.h>
#include <nfc/nfc_device.h>
#include <nfc/nfc_poller.h>
#include <nfc/protocols/st25tb/st25tb.h>
#include <nfc/protocols/st25tb/st25tb_poller.h>
#include <dialogs/dialogs.h>
#include <storage/storage.h>
```

The writer skips blocks 5 and 6 (read-only system blocks) and writes all remaining blocks sequentially.

<!-- Add your images to this folder and reference them like: -->
<!-- ![screenshot](screenshot.png) -->
