---
title: "tGIF"
summary: "P3rf3ctr00tCTF misc chalenge"
categories: ["P3rf3ctr00tCTF","Blog",]
tags: ["P3rf3ctr00tCTF"]
#externalUrl: ""
#showSummary: true
date: 2024-11-23
draft: false
---


# **Challenge Description**

- **Name:** tGIF
- **Category:** misc
- **Points:** 50
- **Challenge Description:**  
    TGIF, but I think I'm either rendering my file wrongly or the dimensions are just off.

---

# File Analysis

The challenge began with a file named `tgif` that did not have a recognizable file type. Running the `file` command gave the following output:
```shell
└─$ file tgif        
tgif: data
```

The `file` utility could not identify the file type. This hinted that the file might have corrupted or missing magic headers.
## Hexadecimal Analysis

To investigate further, I opened the file in `hexedit` and examined the first few bytes. Here is part of the header:
```mathematica
00000000   47 04 46 38  39 61 90 01  C8 00 F7 00  00 00 00 00  G.F89a..........
00000010   25 1E 25 3A  31 34 3D 43  62 46 3E 48  48 35 32 4E  %.%:14=CbF>HH52N 
00000020   4D 65 4F 3A  34 56 4F 61  57 45 46 5A  3F 38 5E 46  MeO:4VOaWEFZ?8^F
```

The header did not match any known magic number for standard file types. However, parts of it (`47 04 46 38 39 61`) resembled the magic header for GIF files, which should be `GIF89a`. [GIF Bits and Bytes guide](https://giflib.sourceforge.net/whatsinagif/bits_and_bytes.html#image_descriptor_block) 
### Correcting the Magic Header

I replaced the incorrect bytes (`47 04`) with `47 49` (`G` and `I`) to create the proper magic header `GIF89a`. This allowed the file to be recognized as a GIF image.
```
└─$ file tgif 
tgif: GIF image data, version 89a
```

## GIF Integrity Check

After fixing the header, I opened the GIF. It was partially corrupted—the image displayed, but it was incomplete. This suggested that some other fields in the header, such as width or height, were incorrect.
![gif1](https://gist.github.com/user-attachments/assets/7742113a-b8de-4803-9f51-e728a4768fa2)

### Modifying the Height

To fix the GIF, I adjusted the height value in the header. I used the online tool [RedKetchup GIF Resizer](https://redketchup.io/gif-resizer) to modify the height and correct the file.
![gif2](https://gist.github.com/user-attachments/assets/47d123ad-a051-4650-9975-62237c81b165)

## Flag

After correcting the height, the GIF displayed correctly, revealing the full image showing the flag

`r00t{d38252762d3d4fd229faae637fd13f4c}`

