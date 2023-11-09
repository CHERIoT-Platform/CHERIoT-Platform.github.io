---
layout: post
title:  "CHERIoT Programmers' Guide drafts online"
date:   2023-11-06T15:49+00:00
categories: guide documentation
---

You might have noticed a link in the header to the [CHERIoT Programmers' Guide](https://cheriot.org/book).
This is an early public draft and will be written in in full view of a live audience of tens (optimistically) of avid readers!

This is using [AsciiDoxy](https://asciidoxy.org), which can parse Doxygen XML and generate AsciiDoc output, to include documentation for the CHERIoT RTOS APIs.
Doxygen, in turn, can use libclang for parsing.
The [book build container](https://github.com/CHERIoT-Platform/book/pkgs/container/book-build-container) contains a version of Doxygen built with the version of libclang from the CHERIoT compiler, which can then recognise the CHERIoT-specific attributes.
This lets it directly reference the RTOS source and should make it easy to update as the code evolves.

Currently, all of the cross references that go between chapters are broken in the HTML version and there are likely to be a lot of other errors (typographical, factual, and grammatical).
This is a live document and will evolve over time, hopefully leading to fewer errors both in absolute numbers and as a proportion of the total text.
