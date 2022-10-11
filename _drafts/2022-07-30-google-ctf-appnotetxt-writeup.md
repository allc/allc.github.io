---
layout: post
title: Google CTF 2022 APPNOTE.TXT Writeup
tags: [ctf, ctf writeup]
---

*Note: Some of the links to external sites about the challenge in this post might become unavailable in the future.*

## The Challenge

APPNOTE.TXT is a [challenge](https://capturetheflag.withgoogle.com/challenges/misc-appnote) under the misc category at [Google CTF](https://capturetheflag.withgoogle.com/) 2022. It was solved by 210 teams out of 382 teams on the [scoreboard](https://capturetheflag.withgoogle.com/scoreboard/) with at least one challenge solved.

Challenge description:

> Every single archive manager unpacks this to a different file...

The challenge also has an [attachment](https://storage.googleapis.com/gctf-2022-attachments-project/2551253642bde3066e55c9cc8e9b0b4aa77feadc00c81032da778e6f7c89907135dfc2611fd8617204720dbfadb31429ae11f6ecd202887f4ce99f2f53a3c5e8), which contains a ZIP file named `dump.zip`.

## First Investigations

In the description, it says every archive manager unpacks this to a different file. Normally it means that the ZIP file is malformed, e.g. if there is a length in the metadata, it does not match the actual content of the file etc., and different implementations of the archive managers will handle it differently. However, later I find this is a bit misleading, but it does suggest the ZIP file is "interesting" (well, if the file is the only thing in this challenge, besides the very short description, it must be...).

I tried what it suggested in the challenge description, which is to try to open it with different archive managers. I tried using File Explorer in Windows 11, 7-Zip, PeaZip and Gzip. However, they all unpack the file the same way, got a single text file named `hello.txt` with the content

> There's more to it than meets the eye...

There must be more things in this file, and I ran `binwalk` on this file. From the output of `binwalk`, I could see that there are more Zip archive data contained in this file, and their names are `hello.txt`, `hi.txt`, `flag00` to `flag18`, with each of the "flag" files having multiple appearances.

I didn't yet know what information these files could give, but let me extract them first. The way I did was
