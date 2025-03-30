---
layout: post
title: "The Making of Rift: Part 3, Multi-Controller Networking"
---

## Introduction

In the last blog post, I talked about how I integrated four-player multiplayer into my raycaster. Now, I will tell the tale of my adventures in multi-controller networking.

## Multi-Controller

While perusing the datasheet, I found out that it was possible to have more than one controller on the bus. Combined with my previous realisation of roles not being set in stone, this would make everything so much simpler! Instead of relaying data around, each device could become a controller to send their data via general calls and then become a target in the meantime, using just one callback to receive. 


## Initial Problems
My first implementation was exactly as described. The handshaking was changed ever slightly to now use `id` and assign everyone an address.  And it worked amazingly!
```cpp
void rx_event(uint8_t *buffer, int length) {
    if (role != buffer[0]) {
        sprites[buffer[0]] = *(sprite_t *)buffer;
    }
}
void loop() {
	... 
	twi_writeTo(0x00, (uint8_t *)&sprites[id], sizeof(sprite_t), false, true);
	...
}
```
However, after a couple minutes, everything would freeze. Looking on my logic analyser, the bus would just hang and lock up forever. The Wire library core had timeouts, but that seems like solving it by covering it up. I would have to go deeper.

## I2C's Inferno

The problems seemed to arise from failing arbitration; that is, preventing one controller from using the bus when another is using it. The hardware had systems in place to prevent this, but they didn't seem to be working perfectly.

From the datasheet:
```
 An algorithm must be implemented allowing only one of the masters to complete the transmission. All 
other masters should cease transmission when they discover that they have lost the selection process. 
This selection process is called arbitration. When a contending master discovers that it has lost the 
arbitration process, it should immediately switch to Slave mode to check whether it is being addressed by the winning master. The fact that multiple masters have started transmission at the same time should not be detectable to the slaves, i.e. the data being transferred on the bus must not be corrupted.

...

The wired-ANDing of the bus lines is used to solve both these problems. The serial clocks from all master will be wired-ANDed, yielding a combined clock with a high period equal to the one from the Master with the shortest high period. The low period of the combined clock is equal to the low period of the Master with the longest low period. Note that all masters listen to the SCL line, effectively starting to count their SCL high and low time-out periods when the combined SCL line goes high or low, respectively.

Arbitration is carried out by all masters continuously monitoring the SDA line after outputting data. If the value read from the SDA line does not match the value the Master had output, it has lost the arbitration. Note that a Master can only lose arbitration when it outputs a high SDA value while another Master outputs a low value. The losing Master should immediately go to Slave mode, checking if it is being addressed by the winning Master. The SDA line should be left high, but losing masters are allowed to generate a clock signal until the end of the current data or address packet. Arbitration will continue until only one Master remains, and this may take many bits. If several masters are trying to address the same Slave, arbitration will continue into the data packet.
```
I scoured the internet to see if anyone had come across this problem. At first, I only found things like this:
```
Call me a coward if you will, but my solution to multi-mastering and clock-stretching issues is to avoid them. And that means using multiple I2C bus networks. By far the simplest way to do so is to buy a microcontroller that has more than one hardware I2C controller built in.
```
from [this article](https://hackaday.com/2016/07/19/what-could-go-wrong-i2c-edition/).

But I wasn't ready to give up yet. I spent several months endlessly trying new things without success. It seemed almost hopeless, that there was an unsolvable hardware bug and that multi-controller was a dream without reality.

Finally, when it seemed like all hope was lost, I came across [this article](https://www.robotroom.com/Atmel-AVR-TWI-I2C-Multi-Master-Problem.html). Another person had found my same bug! You should definitely read the article, but I'll try to summarize it. Basically, what was happening was when the device's code was processing the interrupt triggered when there is a STOP on the I2C bus, it did not notice another device's message had started in the meantime. So, after it had finished processing, it thought the bus was clear and started to send another message. Sending two messages at the same time is not a great idea, and so both controllers fight, locking up the bus for everyone else and never actually finishing.

The article posed two solutions; the first, checking if the SDA line and SCL line was low at the same time, a guarenteed busy bus, before sending anything, or, one that the author did not try, rewriting the whole interrupt in assembly code to squeeze the last drop of speed out of it so the device would not be stuck in the interrupt for so long.

Being of an inquisitive nature, I tried the latter and completely re-wrote the entire I2C library from scratch with a full assembly ISR (interrupt service routine). I added a special condition so that the stop interrupt was checked for before anything else, like preserving the rest of the registers, was done. I also added my handshaking for more than two players.


Unfortunately, much to my dismay, it didn't solve the problem. However, I did get a super-optimized ISR out of it. I did remove the early checking for the stop because it was overcomplicating the code. I tried the former solution (with some slight adjustments), and it worked perfectly! Finally, I could send multi-controller messages for days at a time without hanging the bus. After a long odyssey, I finally got it working.

The beginning of the ISR (without the prologue):
```assembly
; set up Y pointer (data)
ldi r28, lo8(%[data])
ldi r29, hi8(%[data])

; set up Z pointer (TW registers)
ldi r30, TWPTR
clr r31

; switch (TWSR)
ldd r18, Z + TWSR ; no mask needed because prescaler bits are cleared

cpi r18, 0x08
breq TW_START

; MT_MR
cpi r18, 0x18
breq TW_MT_SLA_ACK
cpi r18, 0x28 
breq TW_MT_DATA_ACK
cpi r18, 0x38
breq TW_MT_ARB_LOST ; same as TW_MR_ARB_LOST
cpi r18, 0x40
breq TW_MR_SLA_ACK
cpi r18, 0x50
breq TW_MR_DATA_ACK
cpi r18, 0x58
breq TW_MR_DATA_NACK

; 64 instruction limit on branches
rjmp SR_ST 

TW_START:
    ; TWDR = i2c_detail::data.slaRW;
    ldd r26, Y + %[slaRW]
    std Z + TWDR, r26
    ; TWCR = REPLY_NACK;
    ldi r26, REPLY_NACK
    std Z + TWCR, r26
    ; return;
    rjmp pop_reti

TW_MT_SLA_ACK:
TW_MT_DATA_ACK:
    ; if (i2c_detail::data.bufferIdx >= i2c_detail::data.bufferSize) { stop(); return; }
    ldd r26, Y + %[bufferIdx]
    ldd r27, Y + %[bufferSize]
    cp r26, r27
    
    brlt 1f ; 64 instruction limit on branches
    rjmp stop_reti
    1:

    ; TWDR = i2c_detail::data.twiBuffer[i2c_detail::data.bufferIdx++];
    inc r26
    std Y + %[bufferIdx], r26

    ; Use SUBI and SBCI as (non-existant) ADDI and (non-existant) ADCI
    ; bufferIdx is already incremented so decrement to compensate

    clr r27
    subi r26, lo8(-(%[twiBuffer] - 1))
    sbci r27, hi8(-(%[twiBuffer] - 1))
    ld r26, X
    std Z + TWDR, r26

    ; TWCR = REPLY_NACK;
    ldi r26, REPLY_NACK
    std Z + TWCR, r26
    ; return;
    rjmp pop_reti

TW_MT_ARB_LOST:
    ; TWCR = REPLY_ACK;
    ldi r26, REPLY_ACK
    std Z + TWCR, r26
    ; i2c_detail::data.error = TW_MT_ARB_LOST;
    ldi r26, 0x38
    std Y + %[error], r26
    ; active = false;
    ; return;
    rjmp active_false_reti
; ----------------------------------------------------- ;
```

```cpp
uint8_t busyChecks = I2C_BUS_BUSY_CHECKS;
while (busyChecks) {
	if ((I2C_SCL_PIN & _BV(I2C_SCL_BIT)) && (I2C_SDA_PIN & _BV(I2C_SDA_BIT))) {
		busyChecks--;
	} else {
		i2c_detail::data.error = TW_MT_ARB_LOST;
		return;
	}
}
```

I released my library I had written with the multi-controller fixes, handshaking for multiple players, and assembly ISR [here](https://community.arduboy.com/t/arduboyi2c-i2c-library/12278?u=sublinear).

Now, the code looks like this:
```cpp
void on_receive() {
    uint8_t *buffer = I2C::getBuffer();
    uint8_t id = ((sprite_t *)buffer)->id;
    uint8_t *sprite = (uint8_t *)&sprites[id];
    for (uint8_t i = 0; i < sizeof(sprite_t) - 2; i++) { // copy all but id and type
        sprite[i] = buffer[i];
    }
    sprites[id].timeout = 1;
}
void update_multiplayer() {
    // send data over I2C
    I2C::write(0x00, &sprites[id], false);
}
```
Ahh. Much better.