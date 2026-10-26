---
title: "ST25TB Writer for Flipper Zero"
date: 2026-04-06
draft: true
tags: ["flipper-zero", "nfc", "st25tb"]
summary: "Building a Flipper Zero application to read, edit, and write ST25TB tags directly on the device"
---

## Introduction

After getting a Flipper Zero, I started experimenting with the NFC (Near Field Communication) tags I had around. Most of them could be read and written using the tools already available, but one family behaved differently: ST25TB. The official NFC application could identify the tag, read its memory, and save it to an `.nfc` file, but it did not offer a way to write that data back.

While researching the protocol, I found Marco Zoratti's article, [*Reverse Engineering: How I hacked vending machines keys with Flipper Zero*](https://marco-zora.medium.com/how-i-hacked-nfc-st25tb-keys-with-flipper-zero-d1a3b1833cf2). His analysis of ST25TB tags and of the information exposed by the Flipper firmware was a useful starting point for this project. His work focuses on understanding a specific vending-machine key; my goal was narrower: build a small, reusable tool for reading, editing, and writing ST25TB memory directly from the Flipper.

At the time I created the application, I could not find a ready-to-use, publicly available Flipper Zero application that did this. The firmware contained the protocol support required to read and write individual blocks, but there was no user-facing workflow for restoring data from a dump. That missing piece became the reason for building **ST25TB Writer**.

> This project is intended for tags you own or are explicitly authorized to test. Writing arbitrary data to access, ticketing, or payment systems can damage the tag and may be illegal.

## Inside an ST25TB tag

Before writing a single byte, I wanted to understand what I was actually holding. ST25TB is a family of contactless memory tags from STMicroelectronics that run at 13.56 MHz over the ISO (International Organization for Standardization) 14443 Type B interface with a small proprietary command set. The interesting part is not the protocol, though — it is that the tag is *passive*: it only exists, electrically, while a reader is powering it. Almost everything surprising about these tags, including the attack later in this post, follows from that one fact, so it is worth looking at the hardware first.

### The radio-frequency field is both power and communication

![A Flipper Zero reader and a passive ST25TB04K tag communicating wirelessly: power and requests travel toward the tag, while replies travel back to the reader.](tag-overview.svg)

An ST25TB tag has no battery or other independent power source. The reader — here, the Flipper Zero — generates a 13.56 MHz field. When the tag enters that field, it receives the energy needed to operate and can communicate with the reader. Energy travels from the reader to the tag, while information travels in both directions.

The reader sends commands by modulating the field. The tag replies without generating a carrier of its own: it changes the electrical load on its antenna, and the reader detects that variation through inductive coupling.

The first diagram deliberately treats the tag as a black box. Opening that box gives the main components documented in the [ST25TB04K datasheet](https://www.st.com/resource/en/datasheet/st25tb04k.pdf):

![The main components inside an ST25TB04K: antenna, power supply, ASK demodulator, digital control, EEPROM, and BPSK load modulator.](tag-internals.svg)

Each component has one job:

- **Antenna:** receives the RF (Radio Frequency) energy and the signal carrying the reader's request. It is also the path through which the tag sends its reply.
- **Power supply:** converts the energy collected by the antenna into the voltage needed by the rest of the chip.
- **ASK (Amplitude-Shift Keying) demodulator:** observes the amplitude of the received carrier. The reader encodes its request by changing that amplitude, so the demodulator removes the carrier and reconstructs the stream of request bits.
- **Digital control:** reconstructs and validates the request frame, checks its CRC (Cyclic Redundancy Check), decodes the command and its parameters, and then starts the corresponding memory operation. It also prepares any data that must be returned to the reader.
- **EEPROM (Electrically Erasable Programmable Read-Only Memory):** stores the tag's persistent data. We will open this block in the next section.
- **BPSK (Binary Phase-Shift Keying) load modulator:** receives the reply bits from the digital control and changes the load on the antenna accordingly, allowing the reader to recover the response.

The two blocks that matter most for the rest of the post are the digital control, where a request becomes a memory operation, and the EEPROM, where that operation physically takes place. We will expand both of them later.

The important consequence is that communication and power are not independent. Turning off the reader's field also cuts the tag's power. Once the internal voltage drops too far, the logic and any ongoing EEPROM programming stop. That is the mechanism we will use later in the tear-off experiment.

### Opening the EEPROM

![The logical ST25TB04K memory map, with one 32-bit block expanded into its 32 individual bits.](memory-map.svg)

From the protocol's point of view, the EEPROM is a flat array of 32-bit blocks: every `Read_block` or `Write_block` command moves exactly four bytes. The ST25TB04K has 4 Kbit of user EEPROM, giving 128 addressable blocks. Their behavior is not uniform:

- **Blocks 0–4** — a resettable OTP (One-Time Programmable) area. Individual bits can move from 1 to 0, while a dedicated command resets the entire area to 1.
- **Blocks 5 and 6** — two 32-bit counters that behave unlike anything else on the tag.
- **Blocks 7–15** — user memory that can be permanently locked through the system area.
- **Blocks 16–127** — ordinary user memory.
- **Block 255** — the system area containing OTP and lock bits.

The 64-bit UID (Unique Identifier) is different: it is factory-programmed in a separate ROM (Read-Only Memory), returned by `Get_UID`, and is not part of this EEPROM array.

Each block contains 32 bits, and each bit ultimately corresponds to a memory cell. The lower part of the diagram stops there deliberately: the datasheet specifies the logical organization and the behavior of the commands, but not the internal architecture or the physical arrangement of those cells.

A `Read_block` returns the four bytes stored at the requested address. For blocks 7–127, a `Write_block` starts a self-timed programming cycle that includes an automatic erase before the new value is written. According to the [ST25TB04K datasheet](https://www.st.com/resource/en/datasheet/st25tb04k.pdf), this takes about 5 ms under typical conditions. That documented sequence is all we need before looking at what happens to a single bit during a write.

Blocks 5 and 6 are the exception. The hardware accepts a new counter value only if it is **lower** than the current one: they are **decrement-only**, meant for values that must never increase. They also have extra anti-tearing logic to preserve a valid value if power disappears during an update. A normal writer therefore cannot increment them. Why that guarantee can be disturbed is a question about the cells themselves.

### One bit is stored as charge

![A conceptual floating-gate EEPROM cell and its read threshold: an erased cell has a low threshold and reads as 1, a programmed cell has a high threshold and reads as 0, while an interrupted operation can leave a weak bit close to the reference.](eeprom-cell.svg)

The ST25TB04K datasheet does not reveal the chip's memory process or transistor-level layout. The following explanation therefore uses the conventional floating-gate EEPROM model to explain the observed behavior; the drawing is conceptual, not a reconstruction of the ST25TB04K die.

In that model, a cell is a floating-gate MOS (Metal-Oxide-Semiconductor) transistor. In addition to the control gate used to access it, it contains a second gate completely surrounded by insulating oxide. Electrons trapped on this **floating gate** cannot easily escape when power is removed, which is why the tag remembers its contents while lying unpowered in a wallet.

The stored charge changes the transistor's threshold voltage, usually written as Vₜ. Reading a cell does not measure the electrons directly. The circuit applies a known voltage to the control gate and observes whether current can flow through the channel between source and drain. The sense amplifier compares that response with a reference and emits a digital bit:

- An **erased** cell has a lower Vₜ, conducts under the read condition, and is interpreted as **1**.
- A **programmed** cell contains additional trapped electrons. Its Vₜ is higher, so it does not conduct under the same condition and is interpreted as **0**.

Erase and program operations move charge gradually rather than instantaneously. Cutting power during the process can leave the cell close to the read reference instead of safely on either side. The result is a **weak bit**: a cell whose physical state is neither a robust 0 nor a robust 1 and whose observed value may change with coupling distance, supply voltage, temperature, or the characteristics of that particular cell.

This erase-then-program sequence and its intermediate states are described in more detail in the research paper [*Tears for Fears, Breaking an RFID Counter*](https://www.sstic.org/media/SSTIC2024/SSTIC-actes/tears_for_fears_breaking_an_rfid_counter/SSTIC2024-Article-tears_for_fears_breaking_an_rfid_counter-marty_granier_delion_JHhE6Td.pdf). Under normal use these states remain invisible because the operation is allowed to finish and the counters' anti-tearing logic keeps exposing a valid value. A cell, however, cannot distinguish an accidental loss of power from a field deliberately removed at a precise instant.

Three facts now sit together: the field is the tag's only power source, the counters are protected decrement-only blocks, and the cells beneath them are analog things that can be caught mid-change. That combination is enough to suspect the counters are not as one-way as they look — which is exactly what the tear-off attack exploits.

## Building the writer

The application is built with the official Flipper Zero firmware toolchain and uses the NFC implementation already present in the firmware. This keeps the application focused on the missing workflow rather than reimplementing the RF protocol itself.

From the user's point of view, the process is straightforward:

1. Launch ST25TB Writer and choose whether to read a new tag or open an existing `.nfc` capture.
2. Inspect the tag information and navigate through its memory two blocks at a time.
3. Select either block in a row and edit its four bytes with the Flipper's hexadecimal input screen.
4. Select `OK` when the changes are complete and confirm that they should be written.
5. Place the destination tag on the back of the Flipper while the application writes and verifies only the modified blocks.

<!-- TODO: capture on the Flipper. -->
<!-- ![Choosing between a live tag and a saved capture](choose-source.png) -->

<!-- TODO: capture on the Flipper. -->
<!-- ![Browsing ST25TB information and memory blocks](block-editor.png) -->

<!-- TODO: capture on the Flipper. -->
<!-- ![Editing the four bytes of a selected block](byte-editor.png) -->

<!-- TODO: capture on the Flipper. -->
<!-- ![Writing and verifying the modified blocks](writing-progress.png) -->

### What happens internally

The first screen offers two sources. **Read** starts an ST25TB poller and reads the tag, including its UID, system OTP value, and all regular memory blocks. **Saved** opens the existing Flipper file browser in the NFC directory, accepts `.nfc` files, and rejects captures whose stored protocol is not ST25TB.

The top of the editor contains the information normally found in the NFC application's `Info` view: the detected variant, the UID, and the system OTP value. The memory follows underneath, with two consecutive blocks on every row:

```text
B000  AA AA AA AA  BB BB BB BB
B002  CC CC CC CC  DD DD DD DD
```

The up and down buttons move between rows, while left and right select one of the two four-byte blocks. Pressing `OK` opens the firmware's native hexadecimal input component for that block. A row marked with `*` contains at least one modified block.

The application keeps the original reading as a baseline, so it knows exactly which blocks changed. After the final confirmation, the writer sends only those blocks to the tag. The operation is driven by NFC poller events rather than a blocking loop: each successful write is read back and verified before the application advances to the next modified address. The destination tag's ST25TB variant is also checked before the first write.

Blocks 5 and 6 are available in the editor and are no longer skipped. They still behave as decrement-only counters at the hardware level. The writer also does not attempt to change the UID or the system block at address 255, and OTP or locked blocks remain subject to their normal restrictions.

The resulting state machine is reflected directly in the interface:

- waiting for a tag;
- reading its complete memory or loading a saved capture;
- editing individual blocks;
- writing and verifying the modified blocks;
- reporting completion, an incompatible tag, or the block at which writing failed.

The [source code and build manifest](https://github.com/GATT0/ST25TB-Flipper-Writer) live in a separate repository so that the article can focus on the behavior and design rather than becoming a walkthrough of the C implementation.

### Current limitations

Edits currently live only in memory. Opening a saved capture provides its initial values, but the application does not overwrite or create an updated `.nfc` file. The edited data can be written to a physical tag, then read again with the standard NFC application if a new capture is needed.

The editor covers the normal addressable blocks, including the two counters, but deliberately leaves the UID and system OTP block read-only. It also cannot determine that an apparently writable value will be accepted in every case: OTP transitions, lock bits, and counter rules are enforced by the tag itself. Read-back verification is therefore the final authority on whether a change succeeded.

## Conclusion

ST25TB support was already present in the Flipper Zero firmware, but only part of it was exposed through the standard user interface. Building this application was mostly about connecting the existing file parser, NFC poller, and GUI (Graphical User Interface) into a safe and understandable write workflow.

The result is intentionally small: read a tag or open a previous capture, inspect its complete memory, edit individual bytes, and let the Flipper write and verify only what changed. It does not defeat the hardware protections of ST25TB tags, but it fills the practical gap that originally prevented me from modifying and writing my own tags without preparing a new dump elsewhere.
