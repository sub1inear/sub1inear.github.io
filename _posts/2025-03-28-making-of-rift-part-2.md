---
layout: post
title: "The Making of Rift: Part 2, Advanced I2C Multiplayer"
---

## Introduction

In the last blog post, I talked about the tale of starting out with I2C multiplayer, with networking Pong and integrating it into my raycaster. In this blog post, I'm going to continue the tale and talk about my adventures with four way multiplayer.

## A Switch in APIs

When writing the raycaster code, I had actually switched to extending the Wire library core, `twi.c`. After looking a bit into the code of the Wire library, I was shocked to find out that it had a buffer and then the core had a buffer. To help remove this extreme bloat, I switched to just using the core.

Note: TWI stands for Two Wire Interface and is what the chip calls I2C.

This sparked several things, but the main one was the realisation that the roles of controller and target aren't fixed in stone. The Wire library makes it seem like these are definite roles that shouldn't be changed, but looking at the low level API, you could be a target and assume the role of a controller to send messages, or vice versa.

## Four Way Multiplayer

After I posted my raycaster demo, the creator of the Arduboy, the amazing Kevin Bates, reached out to me and gave me two more Arduboys Minis! A couple days later, they arrived in the mail and I immediately set out to work on a four way multiplayer scheme.

There are two challenges to four way multiplayer: the handshaking and the actual sending of data. I'll start with the first, handshaking.

### Handshaking

Initially, I thought to take the original handshake algorithm and expand it for four players, like so:
```
Become Controller
Request from Target Address 0x10
If there is a Response
	Request from Target Address 0x11
	If there is a Response
		Request from Target Address 0x12
		If there is a Response
			Stay as Controller
			Send Message to 0x10, 0x11, 0x12: Start
			Start
		Else
			Become Target Address 0x12
	Else
		Become Target Address 0x11
Else
	Become Target Address 0x10
```
However, I realized I would be much better off using a loop. What foolishness!

```cpp
volatile uint8_t twi_handshakeState;

void handshakeTxEvent() {
    twi_handshakeState++;
}

uint8_t twi_handshake(const uint8_t *addresses, uint8_t numPlayers) {
    uint8_t role = TWI_CONTROLLER;
    uint8_t dummy;

    for (uint8_t i = numPlayers - 1; i > 0; i--) { // numPlayers-1, ..., 1
        if (!twi_readFrom(pgm_read_byte(&addresses[i]), &dummy, 1, true)) { // if target number i does not exist
            role = i;
            twi_setAddress(pgm_read_byte(&addresses[i]));
            twi_attachSlaveTxEvent(handshakeTxEvent);
            break;
        }
    }

    while (twi_handshakeState < role) {}
    
    return role;
}
```

The role tracks which player we are. Zero is the controller, and everything else is a target.

There's another clever trick in this: catching when it's time to start. I figured out that if I looped from high to low, the number of messages received would equal the role. This was tracked with `twi_handshakeState`. So, all the device would have to do was wait while the `twi_handshakeState` was less than the role.

So, if there were four players, the first would start at role number three. It would receive one message from the second player, one message from the third player, and one message from the fourth player. After the fourth player joined, `twi_handshakeState` would be three. Because it would be equal to the role, it would be time to start! This pattern continues for the second and third player.

If all the roles except zero were taken, it would just end the loop immediately, because the device had done its job triggering all the others to start. It would start because `twi_handshakeState` would be zero and so would the role.

<video controls>
  <source src="https://sub1inear.github.io/assets/images/making-of-rift-part-2/handshake_4.mp4" type="video/mp4">
</video>

### Sending Data, Part 1

Now with the handshaking out of the way, everyone had a role. I thought for a while and came up with of a good way of sending data:
```
If role is controller
	For i from numPlayers to 1
		Request data from player i
		Send data to everyone
	Send controller data to everyone
```
The controller would request/read data from each player, then turn right around and send it out. It would then send its own data.

"How can you possibly send it to everyone?!?" you might be asking. The answer, my friend, lies in general calls.

### A Rude Interuption: General Calls/Broadcasting

While perusing the TWI/I2C section of the datasheet, I stumbled upon a very cool feature: general calls! It allows you to send a message and have everyone receive it. You just set the address to `0x00` and make sure the general call enable bit is set on all devices. This way, if you wanted to send something to everyone, you didn't have to individually send it one at a time to each device.

### Sending Data, Part 2

With this new feature, general calls, the code became much simpler.

```cpp
if (role == TWI_CONTROLLER) {
	for (uint8_t i = 1; i < numPlayers; i++) {
		// get data from target
		twi_readFrom(pgm_read_byte(&targetAddresses[i]), (uint8_t *)&sprites[i], sizeof(sprite_t), true);

		// broadcast it via general call
		twi_writeTo(0x00, (uint8_t *)&sprites[i], sizeof(sprite_t), false, true);
	}
	twi_writeTo(0x00, (uint8_t *)&sprites[TWI_CONTROLLER], sizeof(sprite_t), false, true);
}
```

And it worked!

<video controls>
  <source src="https://sub1inear.github.io/assets/images/making-of-rift-part-2/raycaster_4.mp4" type="video/mp4">
</video>