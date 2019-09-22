---
title: DragonSectorCTF - Reversing Switch Pro's USB keystrokes - PlayCAP 
description: Performing a minimal reversing over Switch Pro's connection over USB in order to simulate pressed buttons
categories: ctf DragonSector
author: Xh4H
tags: wireshark dragonsector ctf switch pro
---

We (TMHC) played DragonSectorCTF - Teaser - and ranked 42, pretty happy about it.


**Challenge description**
```
Miscellaneous, 203 pts
Difficulty: easy (53 solvers)

Here is a recording of the flag being entered into the app.html HTML5 application.
 - PlayCAP.pcapng
 - app.html
```

**Action**
We are given a Packet Capture file (.pcap) and a link to a site. Let's start by accessing the site.

<div style="text-align:center"><img src="/assets/images/pcap_1.png"/></div>

There's a 10x6 map of characters. Inspecting the source, we see an interesting function ``handleButtons``:

<div style="text-align:center"><img src="/assets/images/pcap_2.png"/></div>

It expects 6 different actions:
- right
- left
- up
- down
- reset
- select

``state`` parameter is not used at all there, it just checks that its not empty/undefined/falsy. ``handleButtons("right", "test")`` will move the selected character to the right by one cell.

We also see a few comments that will help us in the future:

<div style="text-align:center"><img src="/assets/images/pcap_3.png"/></div>

``checkChange`` is a defined function that checks the state of certain buttons every 50 miliseconds. After it's declaration we see it's looking for gamePads connected to the computer, therefore we know that a controller with pads was used to type the flag (read the challenge description again). ``X`` button is assigned to "reset" and ``A`` to "select".

Time to open the pcap file.

We are given a network capture containing 5079 packets. After inspecting them we see many packets with the same length, 91. 

We apply the following filter: ``frame.len == 91``. All of these packets have the same data structure, and there is one value we care of: ``Leftover Capture Data``. A quick google search reveals a [ctf writeup](https://medium.com/@ali.bawazeeer/kaizen-ctf-2018-reverse-engineer-usb-keystrok-from-pcap-file-2412351679f4), pretty similar to this challenge.

In order to work with the packet data, we are going to add the ``Leftover Capture Data`` as a column in order to export everything. Right click on a packet's Leftover data and ``Apply as column``.

<div style="text-align:center"><img src="/assets/images/pcap_4.png"/></div>

Now we are ready to export all the data as .csv.

The ``Leftover Capture Data`` looks like this:

<div style="text-align:center"><img src="/assets/images/pcap_5.png"/></div>

## Small summary
- We know that an external controller has been used to type the flag.
- Said controller has a pad, ``X`` and ``A`` buttons.
- This controller's communication has been logged with Wireshark, and the size of each packet is 91 bytes.

After some googling, we find the following [piece of code](https://github.com/ToadKing/switch-pro-x/blob/master/switch-pro-x/ProControllerDevice.cpp#L33):

```cpp
enum {
	SWITCH_BUTTON_USB_MASK_A = 0x00000800,
	SWITCH_BUTTON_USB_MASK_B = 0x00000400,
	SWITCH_BUTTON_USB_MASK_X = 0x00000200,
	SWITCH_BUTTON_USB_MASK_Y = 0x00000100,

	SWITCH_BUTTON_USB_MASK_DPAD_UP = 0x02000000,
	SWITCH_BUTTON_USB_MASK_DPAD_DOWN = 0x01000000,
	SWITCH_BUTTON_USB_MASK_DPAD_LEFT = 0x08000000,
	SWITCH_BUTTON_USB_MASK_DPAD_RIGHT = 0x04000000,

	SWITCH_BUTTON_USB_MASK_PLUS = 0x00020000,
	SWITCH_BUTTON_USB_MASK_MINUS = 0x00010000,
	SWITCH_BUTTON_USB_MASK_HOME = 0x00100000,
	SWITCH_BUTTON_USB_MASK_SHARE = 0x00200000,

	SWITCH_BUTTON_USB_MASK_L = 0x40000000,
	SWITCH_BUTTON_USB_MASK_ZL = 0x80000000,
	SWITCH_BUTTON_USB_MASK_THUMB_L = 0x00080000,

	SWITCH_BUTTON_USB_MASK_R = 0x00004000,
	SWITCH_BUTTON_USB_MASK_ZR = 0x00008000,
	SWITCH_BUTTON_USB_MASK_THUMB_R = 0x00040000,
};
```

These are the masks used for every single button in the Switch Pro. Next step was looking into the packets to see if we could find these values.

<div style="text-align:center"><img src="/assets/images/pcap_6.png"/></div>

The 7th and 8th bytes tell us which button is being pressed:
- 01 would stand for Y
- 02 would stand for X
- 04 would stand for B
- 08 would stand for A

<div style="text-align:center"><img src="/assets/images/pcap_7.png"/></div>
The 11th and 12th bytes tell us which button from the DPAD is being pressed:
- 01 would stand for D
- 02 would stand for U
- 04 would stand for R
- 08 would stand for L

## Time to parse the data

We wrote the following string to parse the dump into actual pressed buttons:
```js
const csv = require("csvtojson");
let chars = [];

let dpad = {
    '01':'D',
    '02':'U',
    '04':'R',
    '08':'L'
}

let buttons = {
	'01': 'Y',
	'02': 'X',
	'04': 'B',
	'08': 'A'
}

let previousButton, previousDpad;


csv()
.fromFile("exported.csv")
.then((jsonObj)=>{
	for (const packet of jsonObj) {
		let switch_packet = packet["Leftover Capture Data"].replace("\"", "");
		let button_type = switch_packet.substr(6, 2); // 7th and 8th byte -> Button type
		let dpad_type = switch_packet.substr(10, 2); // 11th and 12th byte -> dpad type

		if (previousButton != button_type) {
			previousButton = button_type;
			if (buttons[button_type] != undefined) {
				chars.push(buttons[button_type]);
			}
		}

		if (previousDpad != dpad_type) {
			previousDpad = dpad_type;
			if (buttons[dpad_type] != undefined) {
				chars.push(dpad[dpad_type]);
			}
		}	
	}

	require("fs").writeFileSync("parsed.json", JSON.stringify(chars));
})
```
**Important**: Whenever you press a button, be it in a keyboard or a controller, you are sending a lot of packets with the same information, X button being pressed. In order to not have duplicated-useless data, we added the check of ``previousButton`` and ``previousDpad`` so that we don't add more "X button was pressed" until X was in fact released (key up).


With the output, we use as well the following script in order to simulate the pressed buttons directly into the browser:

```js
let goLeft = () => handleButtons('left', 'test');
let goRight = () => handleButtons('right', 'test');
let goUp = () => handleButtons('up', 'test');
let goDown = () => handleButtons('down', 'test');
let goReset = () => handleButtons('reset', 'test');
let goSelect = () => handleButtons('select', 'test');

let arr = ["Y","Y","Y","Y","Y","Y","Y","Y","Y","Y","Y","X","X","X","R","R","R","A","D","D","D","D","A","U","L","A","R","R","R","R","R","R","R","A","R","R","R","R","R","U","U","R","R","R","R","A","D","D","D","D","A","U","U","U","U","L","L","L","L","L","L","L","A","D","D","L","A","R","R","R","R","R","D","A","L","A","U","U","U","R","A","D","D","R","R","R","L","A","U","L","A","D","D","L","L","L","L","L","L","D","A","U","U","U","U","U","A","R","R","R","R","R","R","A","D","D","A","R","R","D","A","R","R","A","R","D","A","U","U","R","R","R","R","R","A","R","R","R","A","D","D","D","A","X"]

function insertKey(letter) {
	switch (letter.toLowerCase()) {
		case "l":
			goLeft();
			break;
		case "r":
			goRight();
			break;
		case "u":
			goUp();
			break;
		case "d":
			goDown();
			break;
		case "x":
			goReset();
			break;
		case "a":
			goSelect();
			break;
		default:
			console.log("Skipping");
	}
}

for (const letter of arr) {
	insertKey(letter.toLowerCase())
}
```

<div style="text-align:center"><img src="/assets/images/pcap_8.png"/></div>

**Flag**: DrgnS{LetsPlayAGamepad}


Thanks for reading :)

<script src="https://www.hackthebox.eu/badge/21439"></script>
