---
layout: post
title: "Introduction"
---

Test of initial post...

## Some source code

A fixed-point reciprocal sqrt for the 8-bit AVR

```cpp
// Computes 1/sqrt(x) for x=[0,0xffff] in Q0
// x > 0 returns 1/sqrt(x) in Q16
// x = 0 returns 0xffff
//
// |error| <= 3.0 ULP
FORCEINLINE uint16_t rsqrt(uint16_t x) {

    if (x <= 1) return 0xffff;

    // normalize
    uint8_t e = norm(x);
    if (e & 0x1) x >>= 1;
    e &= 0xfe;  // e=[0,14], even

    // initial estimate
    uint8_t idx = (x >> 9) - 32;
    uint8_t t = pgm_read_byte(&rsqrt_tab[idx]); // Q8

    // Newton-Raphson iteration
    uint16_t s = t * t;
    uint16_t r = 0xc000 - UMULHI(x, s); // Q16
    r = UMUL32(r, t << 1) >> 8;

    // undo normalize
    e = 7 - (e >> 1);
    return r >> e;  // Q16
}
```
