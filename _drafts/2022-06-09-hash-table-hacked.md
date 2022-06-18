---
layout: post
title: Hash Table Hacked
tags: [algorithms, data structure, hash table, python]
---

This post talks about how hash table works, and it could fail with some crafted input data.

In [Question F in Codeforces Round #790 (Div. 4)](https://codeforces.com/contest/1676/problem/F), my solution in Python 3 which used dictionary passed the test during the contest, however was "hacked" and got "time limit exceeded" result on some input.

## How Hash Table Works

To investigate how this happened, we have a look at how hash table (dictionary in Python 3, and many other programming languages) works.

Taking Python 3.10.4 dictionary as an example, which the look up code can be found at <https://github.com/python/cpython/blob/9d38120e335357a3b294277fd5eff0a10e46e043/Objects/dictobject.c#L795>.
