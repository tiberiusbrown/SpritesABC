# SpritesABC

[SpritesABC](https://github.com/tiberiusbrown/SpritesABC) is a library for the [Arduboy FX](https://www.arduboy.com/) that provides performance-optimized drawing routines for sprites stored on the Arduboy FX's flash chip. The performance is probably overkill for most monochrome games, but can be vital for [grayscale](https://github.com/tiberiusbrown/ArduGray_Demo).

SpritesABC was originally written for the [Arduboy interpreter for ABC](https://github.com/tiberiusbrown/abc).

## Interface

```c++
struct SpritesABC
{
    // color: 0 for BLACK, 1 for WHITE
    static void fillRect(int16_t x, int16_t y, uint8_t w, uint8_t h, uint8_t color);

    static constexpr uint8_t MODE_OVERWRITE      = 0;
    static constexpr uint8_t MODE_PLUSMASK       = 1;
    static constexpr uint8_t MODE_SELFMASK       = 4;
    static constexpr uint8_t MODE_SELFMASK_ERASE = 6; // like selfmask but erases pixels

    static void drawBasicFX(
        int16_t x, int16_t y, uint8_t w, uint8_t h,
        uint24_t image, uint8_t mode);
    
    static void drawSizedFX(
        int16_t x, int16_t y, uint8_t w, uint8_t h,
        uint24_t image, uint8_t mode, uint16_t frame);
    
    static void drawFX(
        int16_t x, int16_t y,
        uint24_t image, uint8_t mode, uint16_t frame);
};
```

- Use `SpritesABC::drawBasicFX` whenever possible.
  - The `image` parameter should point to raw pixel FX data (without width/height information).
- Otherwise, use `SpritesABC::drawSizedFX` if the sprite dimensions are known at call-time to avoid an additional FX seek.
  - The `image` parameter should point to raw pixel FX data (without width/height information).
- Otherwise, use `SpritesABC::drawFX` as a general purpose method.
  - The `image` parameter should point to FX data in `Sprites::drawOverwrite` or `Sprites::drawPlusMask` format.

## Performance and Design Notes

- `SpritesABC` methods require sprite height to be a multiple of eight pixels (this is more restrictive than `Sprites`/`SpritesB` methods or `FX::drawBitmap`).
- For `SpritesABC::drawFX`, the width and height should be encoded in the sprite data as single byte values, as in sprites encoded for `Sprites`/`SpritesB`/`SpritesU`.
- `SpritesABC::drawSizedFX` and `SpritesABC::drawBasicFX` are to be preferred to `SpritesABC::drawFX` when possible. The latter requires an additional seek to retrieve the sprite dimensions, which costs around 100-150 additional cycles per call.
- SpritesABC routines support only FX sprites, not `PROGMEM` sprites, and the setup and clipping calculations for sprite drawing are intermixed with the initial FX seek.
- In most cases SpritesABC uses a [17-cycle SPI receive-only loop](https://community.arduboy.com/t/avr-spi-musings/11949/13) to stream sprite data from the flash chip.
- Like SpritesU, SpritesABC does not perform an FX reseek every eight rows of pixels unless necessary due to clipping.
- `Sprites::drawPlusMask` contains an optimization for when the y-coordinate is a multiple of eight, and is faster than SpritesABC in this case. This optimization is not present in SpritesU and is not feasible for FX sprites where the SPI transfer rate is a performance limiter.

## Performance Benchmarks

The numbers in the following tables are cycles, counted by [Ardens](https://github.com/tiberiusbrown/Ardens) using `break` instructions to denote the beginning and end of the measured section ([benchmark code](https://github.com/tiberiusbrown/Ardens/tree/master/bench/cycles/sprites_cycles)).

```c++
template<class F>
void debug_cycles(F&& f)
{
    uint8_t sreg = SREG;
    cli();
    asm volatile("break\n");
    f();
    asm volatile("break\n");
    SREG = sreg;
}
```

### Dimensions: 2x8

| Method | (0, 0) f0 | (0, 0) f1 | (0, 4) f0 | (-1, 0) f0 |
|---|:-:|:-:|:-:|:-:|
| <td colspan=5>*Unmasked*</td> |
| Sprites::drawOverwrite | 298 | 306 | 374 | 247 |
| SpritesB::drawOverwrite | 421 | 443 | 519 | 358 |
| SpritesU::drawOverwrite | 298 | 322 | 298 | 286 |
| FX::drawBitmap (dbmOverwrite) | 643 | 642 | 656 | 600 |
| SpritesU::drawOverwriteFX | 620 | 644 | 620 | 607 |
| SpritesABC::drawFX (MODE_OVERWRITE) | 431 | 432 | 431 | 423 |
| SpritesABC::drawSizedFX (MODE_OVERWRITE) | 316 | 312 | 311 | 303 |
| SpritesABC::drawBasicFX (MODE_OVERWRITE) | 266 | - | 266 | 260 |
| <td colspan=5>*Masked*</td> |
| Sprites::drawPlusMask | 267 | 291 | 311 | 241 |
| SpritesB::drawPlusMask | 459 | 483 | 597 | 377 |
| SpritesU::drawPlusMask | 324 | 348 | 324 | 304 |
| FX::drawBitmap (dbmMasked) | 666 | 664 | 678 | 613 |
| SpritesU::drawPlusMaskFX | 654 | 678 | 654 | 625 |
| SpritesABC::drawFX (MODE_PLUSMASK) | 464 | 464 | 463 | 435 |
| SpritesABC::drawSizedFX (MODE_PLUSMASK) | 376 | 334 | 333 | 315 |
| SpritesABC::drawBasicFX (MODE_PLUSMASK) | 309 | - | 279 | 270 |

### Dimensions: 8x8

| Method | (0, 0) f0 | (0, 0) f1 | (0, 4) f0 | (-4, 0) f0 |
|---|:-:|:-:|:-:|:-:|
| <td colspan=5>*Unmasked*</td> |
| Sprites::drawOverwrite | 508 | 516 | 794 | 352 |
| SpritesB::drawOverwrite | 787 | 809 | 1119 | 541 |
| SpritesU::drawOverwrite | 418 | 442 | 418 | 346 |
| FX::drawBitmap (dbmOverwrite) | 895 | 894 | 950 | 726 |
| SpritesU::drawOverwriteFX | 734 | 758 | 734 | 664 |
| SpritesABC::drawFX (MODE_OVERWRITE) | 539 | 540 | 539 | 473 |
| SpritesABC::drawSizedFX (MODE_OVERWRITE) | 424 | 420 | 419 | 353 |
| SpritesABC::drawBasicFX (MODE_OVERWRITE) | 374 | - | 374 | 310 |
| <td colspan=5>*Masked*</td> |
| Sprites::drawPlusMask | 411 | 435 | 539 | 313 |
| SpritesB::drawPlusMask | 939 | 963 | 1431 | 617 |
| SpritesU::drawPlusMask | 498 | 522 | 498 | 391 |
| FX::drawBitmap (dbmMasked) | 972 | 970 | 1026 | 766 |
| SpritesU::drawPlusMaskFX | 870 | 894 | 870 | 733 |
| SpritesABC::drawFX (MODE_PLUSMASK) | 674 | 674 | 673 | 540 |
| SpritesABC::drawSizedFX (MODE_PLUSMASK) | 586 | 544 | 543 | 420 |
| SpritesABC::drawBasicFX (MODE_PLUSMASK) | 519 | - | 489 | 375 |

### Dimensions: 16x16

| Method | (0, 0) f0 | (0, 0) f1 | (0, 4) f0 | (-8, 0) f0 |
|---|:-:|:-:|:-:|:-:|
| <td colspan=5>*Unmasked*</td> |
| Sprites::drawOverwrite | 1367 | 1375 | 2493 | 791 |
| SpritesB::drawOverwrite | 2281 | 2303 | 3549 | 1303 |
| SpritesU::drawOverwrite | 908 | 932 | 908 | 596 |
| FX::drawBitmap (dbmOverwrite) | 2021 | 2020 | 2244 | 1348 |
| SpritesU::drawOverwriteFX | 1206 | 1230 | 1206 | 1005 |
| SpritesABC::drawFX (MODE_OVERWRITE) | 991 | 992 | 991 | 797 |
| SpritesABC::drawSizedFX (MODE_OVERWRITE) | 876 | 872 | 871 | 677 |
| SpritesABC::drawBasicFX (MODE_OVERWRITE) | 826 | - | 826 | 634 |
| <td colspan=5>*Masked*</td> |
| Sprites::drawPlusMask | 999 | 1023 | 1463 | 613 |
| SpritesB::drawPlusMask | 2889 | 2913 | 4797 | 1607 |
| SpritesU::drawPlusMask | 1204 | 1228 | 1204 | 749 |
| FX::drawBitmap (dbmMasked) | 2314 | 2312 | 2536 | 1496 |
| SpritesU::drawPlusMaskFX | 1749 | 1773 | 1749 | 1277 |
| SpritesABC::drawFX (MODE_PLUSMASK) | 1530 | 1530 | 1529 | 1064 |
| SpritesABC::drawSizedFX (MODE_PLUSMASK) | 1442 | 1400 | 1399 | 944 |
| SpritesABC::drawBasicFX (MODE_PLUSMASK) | 1375 | - | 1345 | 899 |

### Dimensions: 32x32

| Method | (0, 0) f0 | (0, 0) f1 | (0, 4) f0 | (-16, 0) f0 |
|---|:-:|:-:|:-:|:-:|
| <td colspan=5>*Unmasked*</td> |
| Sprites::drawOverwrite | 4765 | 4773 | 9251 | 2509 |
| SpritesB::drawOverwrite | 8197 | 8219 | 13209 | 4291 |
| SpritesU::drawOverwrite | 2848 | 2872 | 2848 | 1576 |
| FX::drawBitmap (dbmOverwrite) | 6289 | 6288 | 7184 | 3600 |
| SpritesU::drawOverwriteFX | 3062 | 3086 | 3062 | 2143 |
| SpritesABC::drawFX (MODE_OVERWRITE) | 2759 | 2760 | 2759 | 1877 |
| SpritesABC::drawSizedFX (MODE_OVERWRITE) | 2644 | 2640 | 2639 | 1757 |
| SpritesABC::drawBasicFX (MODE_OVERWRITE) | 2594 | - | 2594 | 1714 |
| <td colspan=5>*Masked*</td> |
| Sprites::drawPlusMask | 3327 | 3351 | 5135 | 1789 |
| SpritesB::drawPlusMask | 10629 | 10653 | 18201 | 5507 |
| SpritesU::drawPlusMask | 4008 | 4032 | 4008 | 2161 |
| FX::drawBitmap (dbmMasked) | 7446 | 7444 | 8340 | 4180 |
| SpritesU::drawPlusMaskFX | 5235 | 5259 | 5235 | 3229 |
| SpritesABC::drawFX (MODE_PLUSMASK) | 4922 | 4922 | 4921 | 2952 |
| SpritesABC::drawSizedFX (MODE_PLUSMASK) | 4834 | 4792 | 4791 | 2832 |
| SpritesABC::drawBasicFX (MODE_PLUSMASK) | 4767 | - | 4737 | 2787 |

### Dimensions: 64x64

| Method | (0, 0) f0 | (0, 0) f1 | (0, 4) f0 | (-32, 0) f0 |
|---|:-:|:-:|:-:|:-:|
| <td colspan=5>*Unmasked*</td> |
| Sprites::drawOverwrite | 18281 | 18289 | 35375 | 9305 |
| SpritesB::drawOverwrite | 31741 | 31763 | 50705 | 16123 |
| SpritesU::drawOverwrite | 10173 | 10197 | 10173 | 5253 |
| FX::drawBitmap (dbmOverwrite) | 22888 | 22887 | 26032 | 12135 |
| SpritesU::drawOverwriteFX | 10348 | 10372 | 10348 | 6204 |
| SpritesABC::drawFX (MODE_OVERWRITE) | 9673 | 9674 | 9673 | 5724 |
| SpritesABC::drawSizedFX (MODE_OVERWRITE) | 9558 | 9554 | 9553 | 5604 |
| SpritesABC::drawBasicFX (MODE_OVERWRITE) | 9508 | - | 9508 | 5561 |
| <td colspan=5>*Masked*</td> |
| Sprites::drawPlusMask | 12591 | 12615 | 19391 | 6445 |
| SpritesB::drawPlusMask | 41469 | 41493 | 70673 | 20987 |
| SpritesU::drawPlusMask | 14725 | 14749 | 14725 | 7534 |
| FX::drawBitmap (dbmMasked) | 27501 | 27499 | 30644 | 14443 |
| SpritesU::drawPlusMaskFX | 19112 | 19136 | 19112 | 10585 |
| SpritesABC::drawFX (MODE_PLUSMASK) | 18354 | 18354 | 18353 | 10053 |
| SpritesABC::drawSizedFX (MODE_PLUSMASK) | 18266 | 18224 | 18223 | 9933 |
| SpritesABC::drawBasicFX (MODE_PLUSMASK) | 18199 | - | 18169 | 9888 |

### Dimensions: 128x64

| Method | (0, 0) f0 | (0, 0) f1 | (0, 4) f0 | (-64, 0) f0 |
|---|:-:|:-:|:-:|:-:|
| <td colspan=5>*Unmasked*</td> |
| Sprites::drawOverwrite | 36202 | 36210 | 70384 | 18265 |
| SpritesB::drawOverwrite | 62973 | 62995 | 100881 | 31739 |
| SpritesU::drawOverwrite | 20029 | 20053 | 20029 | 10181 |
| FX::drawBitmap (dbmOverwrite) | 44392 | 44391 | 50672 | 22887 |
| SpritesU::drawOverwriteFX | 20012 | 20036 | 20012 | 11036 |
| SpritesABC::drawFX (MODE_OVERWRITE) | 18825 | 18826 | 18825 | 10300 |
| SpritesABC::drawSizedFX (MODE_OVERWRITE) | 18710 | 18706 | 18705 | 10180 |
| SpritesABC::drawBasicFX (MODE_OVERWRITE) | 18660 | - | 18660 | 10137 |
| <td colspan=5>*Masked*</td> |
| Sprites::drawPlusMask | 24880 | 24904 | 38464 | 12589 |
| SpritesB::drawPlusMask | 82429 | 82453 | 140817 | 41467 |
| SpritesU::drawPlusMask | 29125 | 29149 | 29125 | 14734 |
| FX::drawBitmap (dbmMasked) | 53613 | 53611 | 59892 | 27499 |
| SpritesU::drawPlusMaskFX | 37544 | 37568 | 37544 | 19801 |
| SpritesABC::drawFX (MODE_PLUSMASK) | 36210 | 36210 | 36209 | 18981 |
| SpritesABC::drawSizedFX (MODE_PLUSMASK) | 36122 | 36080 | 36079 | 18861 |
| SpritesABC::drawBasicFX (MODE_PLUSMASK) | 36055 | - | 36025 | 18816 |
