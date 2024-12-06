# Zoom65 v3 Reverse Engineering

## PCB

MCU: `HFD22001A` (Huafenda-rebranded WestBerryTech `WB32FQ95`)

- [MCU Docs](./docs/EN_RM2905025_WB32FQ95xx_V01.pdf)
- [MCU Docs Alt](./docs/EN_DS1905020_WB32FQ95xC_V01.pdf)

### Firmware

- [1.6mm v1.10 firmware](./firmware/ZOOM65V3_UK_via_v1_10_20241008.bin)
- [1.6mm v1.10 via json](./layout/ZOOM65_v3_uk_via_v1.01_20240827.json)

Firmware byte offset: `0x08000000`

SVD: `todo: source or extract a proper svd`

Open sourced firmware references:
- https://github.com/qmk/qmk_firmware/tree/master/keyboards/monsgeek/m1 (same MCU is used in this board, wired only)
- https://github.com/WestberryTech/qmk_westberry (vendor fork with default branch named tri-mode, promising, has bt drivers we might be able to use)

## Screen Module

### HID/Via Commands

All screen module commands are prefixed by `[0x88, payload_len]` and are paddded to 32 bytes.

#### Screen Position

resetting screen back to meletrix logo:
```
prefix . [165, 1, 255]
```

set the screen theme (also resets back to logo)
```
theme_blue = 1
theme_pink = 2

prefix . [165, 1, 255, theme (u8)]
```

moving the screen up one position
```
[165, 0, 34]
```

moving the screen down one position
```
[165, 0, 33]
```

switching the screen to the next page
```
[165, 0, 32]
```

#### Media

deleting the currently uploaded image and reset back to the chrome dino
```
[165, 2, 224]
```

deleting the currently uploaded gif and reset back to nyan cat
```
[165, 2, 225]
```

signaling the start of an upload

> Note: Images *must* be encoded as [rgb565 (2 bytes), 0xff] ([example](./dumps/rgba5658-image.bin))
>       In an ideal world, we wouldn't send an extra 12KiB (1/3) of data that is hardcoded,
>       the screen firmware could have just inserted it if required.
>       Or, the image could be encoded as a single frame, nonrepeating gif, and the decoder
>       logic could have been reused for the static image, allowing the image to be compressed
>       and uploaded even faster, since the screen natively supports LZW compression.

- Example re-encoded gif from web driver: ./dumps/web-driver-cropped-gif.gif
- Example image blob: ./dumps/rgba5658-image.bin

```
channel_image = 0 # must be 110x110
channel_gif   = 1 # must be 111x111 ?

[165, 2, 240, channel (u8)]
```

signaling the length of an upload
```
[165, 2, 208, len (big endian u32)]
```

computing crc hash
```
const A: isize = 4294967295;
let mut val = A;
for byte in data {
    val ^= (*byte as isize) << 24;
    for _ in 0..8 {
        if val & 2147483648 != 0 {
            val = (val << 1) ^ 4374732215;
        } else {
            val <<= 1;
        }
        val &= A;
    }
}

// output hash
[
    (val >> 24 & 255) as u8,
    (val >> 16 & 255) as u8,
    (val >> 8 & 255) as u8,
    (val & 255) as u8,
]
```

sending upload payloads

> Note: The crc hash must be 32 bit aligned inside the payload.
>       The only case you actually need to do this is for the last payload
>       of a gif, since images are always 36300 bytes long, already aligned.

```
[up to 24 bytes, crc hash (u32)]
```

signaling the end of an upload
```
[165, 2, 241, 1]
```

#### Time

Set time
> Year should ideally be truncated to the closest century (not done properly on web driver - it will have an error after 2255)
```
[165, 1, 16, year (u8), month (u8), day (u8), hour (u8), minute (u8), second (u8)]
```

#### Weather

Weather Icons
```
DayClear          = 0,
DayPartlyCloudy   = 1,
DayPartlyRainy    = 2,
NightPartlyCloudy = 3,
NightClear        = 4,
Cloudy            = 5,
Rainy             = 6,
Snowfall          = 7,
Thunderstorm      = 8,
```

Set weather
[165, 1, 32, icon (u8), current (u8), low (u8), high (u8)]

#### System Info

Download rate uses a kind of dumb float 16 encoding, with a min of 0.00, and a maximum possible value of 655.34.

> In an ideal world, this would be a f32, or even just 7 raw u8 numbers to display on each position. IEEE f16 could also be used with the same byte size. But there is plenty of space left for a bigger more accurate number.

```rust
if float <= Self::MIN_F32 {
    return u16::MIN
}
if float >= Self::MAX_F32 {
    return u16::MAX;
}

let mut n = 0u16;
let mut current = 327.68f32 * 2.0;
for _ in 0..16 {
    n <<= 1;
    current /= 2.0;
    if float >= current - 0.005 {
        float -= current;
        n |= 1;
    }
}

// n now contains our encoded number
```

Set system info
```
[165, 1, 64, cpu_temp (u8), gpu_temp (u8), download (DumbFloat16)]
```


