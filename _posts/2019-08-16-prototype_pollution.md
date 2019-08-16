---
title: RedpwnCTF - Performing a prototype pollution attack
description: Writeup for RedpwnCTF blueprint web challenge which involved the usage of a prototype pollution attack
categories: js
author: Xh4H
tags: javascript ctf redpwnctf blueprint prototype pollution mitigations
---

# Introduction
Writeup for RedpwnCTF blueprint challenge which involved ``prototype pollution`` attack.

First of all, what is a ``prototype pollution`` attack? As the name says, it is about polluting the prototype of a base object which can allow us to modify any existing object and get RCE.

In JavaScript, an Array is a base object. These base object have a property which is ``constructor`` and inside of this there is ``prototype``. If someone managed to edit this property, any Array would be affected.

## Challenge description

All the haxors are using blueprint. You created a blueprint with the flag in it, but the military-grade security of blueprint won't let you get it!

[blueprint.tar.gz](https://github.com/Xh4H/xh4h.github.io/blob/master/CTF_Files/RedpwnCTF/blueprint.tar.gz)

## Action

We are given two files, ``package.json`` and ``blueprint.js``. We look at the package:
```json
{
  "name": "blueprint",
  "version": "0.0.0",
  "private": true,
  "dependencies": {
    "lodash": "4.17.11",
    "mustache": "^3.0.1",
    "raw-body": "^2.4.1"
  }
}
```
Here we can see the versions of the dependencies that the ``blueprint.js`` is using. Will be useful for the future.

After examining ``blueprint.js``, which is an HTTP server, I find a strange comment in the code:

<div style="text-align:center"><img src="/assets/images/protopollution1.png" /></div>

It is using ``_.defaultsDeep``, and ``_`` is declared to be as lodash ``const _ = require('lodash')``

A google search shows that lodash has some serious vulnerability affecting all the versions prior to ``4.17.11`` including this version as well. You can read about this finding here: [Snyk research team discovers severe prototype pollution security vulnerabilities affecting all versions of lodash](https://snyk.io/blog/snyk-research-team-discovers-severe-prototype-pollution-security-vulnerabilities-affecting-all-versions-of-lodash/). Lodash package maintainer already fixed this as of today, but many projects are using old versions of lodash, making them vulnerable to this attack. Let's start our attack.

This challenge allows us to create blueprints with a content and they can either be private or public. Every time we access the page without cookies, we get assigned a user_id and a private blueprint is created which contains the flag, but it is set as private. ``/`` endpoint allows us to list all existing **public** blueprints.

<div style="text-align:center"><img src="/assets/images/protopollution2.png"/></div>

``makeId`` function creates a 16 chars long random character, we are not supposed to bruteforce this.

I set up a basic environment with lodash. At my first attempt I couldn't make this vulnerability work because when setting it up I installed lodash like this: ``npm i lodash``, which downloaded the latest version which had fixed this vulnerability. So I removed it ``npm remove lodash`` and installed a vulnerable version, in fact, the one this server is using: ``npm i lodash@4.17.11``. Now it worked :D

In the following screenshot you can see a piece of code I wrote as a Proof of Concept (PoC) where I pollute Array's prototype. As you can see, accessing the ``public`` property of a previously declared Array and as well a new Array declared after the prototype pollution will both return ``true``, seeing that our test was successful it is time to get our flag.

In other words, any Array has now the ``public`` property set to true.
<div style="text-align:center"><img src="/assets/images/protopollution4.png"/></div>

## Time to get the flag
Open up BurpSuite and grab the request of creating a new blueprint, send the payload we previously tested.

<div style="text-align:center"><img src="/assets/images/protopollution3.png"/></div>

Go to ``/`` and boom, flag.

First blueprint is the "private" flag and second is the new created blueprint which we used to pollute the prototype :)

<div style="text-align:center"><img src="/assets/images/protopollution5.png"/></div>

``flag{8lu3pr1nTs_aRe_tHe_hiGh3s1_quA11tY_pr0t()s}``


## Why does this work and mitigations
Lodash module, and many other npm modules ([this](https://github.com/HoLyVieR/prototype-pollution-nsec18/blob/master/paper/JavaScript_prototype_pollution_attack_in_NodeJS.pdf) pdf analyzes deeply this threat and lists vulnerable modules) that perform deep Array copies can allow an attacker to modify object prototypes and take control of that object's properties.

If you are using lodash, simply update it.
If you are performing deep copies or using any affected module, this can be fixed rather easily. The ECMAScript standard version 5 introduced a very interesting set of functionality to the JavaScript language, ``Object.freeze``. When that function is called on an object, any further modification on that object will silently fail. Since the prototype of ``Object`` is an object, it's possible to freeze it. Doing so will mitigate almost all the exploitable case. 

```js
1. Object.freeze(Object.prototype);
2. Object.freeze(Object);
3. ({}).__proto__.test = 123;
4. ({}).test; // this will be undefined
```

Thanks for reading :)

<script src="https://www.hackthebox.eu/badge/21439"></script>

Sources:
https://github.com/HoLyVieR/prototype-pollution-nsec18/blob/master/paper/JavaScript_prototype_pollution_attack_in_NodeJS.pdf
https://snyk.io/blog/snyk-research-team-discovers-severe-prototype-pollution-security-vulnerabilities-affecting-all-versions-of-lodash/
https://snyk.io/vuln/SNYK-JS-LODASH-450202
