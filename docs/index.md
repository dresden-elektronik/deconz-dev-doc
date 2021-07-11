---
title: Introduction
summary: A brief description of my document.
date: 2021-07-11

---
# Introduction

<mark>Work in Progress</mark>

This documentation describes the deCONZ C++ development around the newer so called *Device code* and Device Description Files (DDF). The discussion takes place in the [deCONZ REST-API Plugin V2](http://github.com/dresden-elektronik/deconz-rest-plugin-v2) repository on GitHub.

## Classic REST-API code

The classic REST-API code handles sensors and lights in command handlers which iterate over respective containers. In these loops the code parses commands, often specific to devices based on their model identifier and take further actions.

The ability to recover from errors and faults is limited and often not as robust and coordinated as it could be. The best error handling can be found in IAS and binding code, other parts like executing user commands mostly don't have any error handling at all and carry out commands in a fire and forget approach.

Supporting new devices often requires adding device specific code and quirks in C++, which led to functions going way over a couple of hundrets lines of code, always with the danger of breaking already existing devices and introducing bugs.

## New Device code

The Device code is designed to provide a solution to known issues in the classic REST-API code.

**Features and goals**

- 100&thinsp;% testability for happy paths as well as all errors and faults which might happen.
- Well structured modular code with minimal dependencies betweend modules.
- The C++ part shall know as little as possible about specific devices.
- Device details are described in Device Description Files (DDF) in JSON format.
- Handling of all errors and faults which might occur.
- Only small functions with max. of 200 lines of code.
- Adding a new device can't break another.

With a code base almost a decade old the Device code can't replace everything at once, therefore it runs in parallel to classic REST-API code and replaces parts of it incrementally.
