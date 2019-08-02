---
title: Python 2.7 input vulnerability
description: Python 2.7 input vulnerability | PoC - /bin/bash shell example
categories: python
by: Xh4H
tags: python, proof of concept, shell, input, vulnerability
---
<link rel="shortcut icon" type="image/x-icon" href="images/favicon-32x32.png">


``input`` function has been used widely. As the name states, it allows a program to ask for user input and save it to a variable, let's have a look at an example:

<div style="text-align:center"><img src="/assets/images/python_input_1.png" /></div>

It is important to note that there exists ``raw_input``, ``int_input`` as well.

A common issue is using ``input`` when you want to receive a number. ``int_input`` should be used instead.

Now, let's go with the "vulnerability". ``input`` function would be equivalent to ``eval(raw_input(prompt))``, therefore we can evaluate code.

Following with the code above, a random number gets generated, and we have 1/10 possibilities to guess it. We could bruteforce until we guess it, **or** use the "eval" utility as follows:

<div style="text-align:center"><img src="/assets/images/python_input_2.png" /></div>

Since python evaluates our input, we can call any function or variable in the current scope.

We can also use this to get a shell. 

<div style="text-align:center"><img src="/assets/images/python_input_3.png" /></div>

Please note how we did not get any print like `Congratulations!` or `Wrong! Try again later :(`, that's because or code got evaluated and opened a shell before following with the code execution.

Interesting, huh?

Thanks for reading :)

<script src="https://www.hackthebox.eu/badge/21439"></script>