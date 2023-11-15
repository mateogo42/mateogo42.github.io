+++
title = 'Goi'
date = 2023-11-10T00:55:09-05:00
draft = false
layout = "simple"
+++
I had to choose a project for one course of my masters degree, so I decided to implement a QOI encoder/decoder
using Go. I called it `goi`, but later realized there was already a repository called exactly the same doing exactly
the same :(. Nevertheless, I decided to leave the name as it was as I'm pretty bad at naming things hehe.

## What is QOI?
QOI stands for Quite OK Image. According to its site, QOI is fast. It losslessly compresses images to a similar 
size of PNG, while offering 20x-50x faster encoding and 3x-4x faster decoding. The file format specification is
only one page long, which makes it quite easy to read, it is also really simple, so it is a nice weekend project
to build.


## QOI File Spec
### Header
A QOI file starts with a 14 byte header containing metadata about the image. 
```go
type QoiHeader struct {
    magic      [4]byte  // magic bytes "qoif"
    width      uint32   // image width in pixels (Big Endian)
    height     uint32   // image height in pixels (Big Endian)
    channels   uint8    // 3 = RGB, 4 = RGBA
    colorspace uint8    // 0 = sRGB with linear alpha, 1 = all channels linear
}
```
Only the `width` and `height` fields are actually used by the encoder/decoder. The other fields are purely informative.

### Data Chunks
After the header, we have a series of data chunks which contains the actual image data. There are 6 kind of data chunks, each identified by an operation name. A running 64 length array (zero initialized) `index` of previously seen pixel values is maintained by the encoder/decoder. 
The encoder/decoder also keeps track of the last visited pixel and a `run` (zero initialized) variable. 
#### QOI_OP_RGB
This chunks consists of 4 bytes, storing the raw RGB values from the pixel. The first byte is the OP id `0xfe` (or `11111110` in binary) following the red, blue and green values from the corresponding pixel.
<pre style="padding:0; display: inline-block;">
┌─ QOI_OP_RGB ────┬─────────┬─────────┬─────────┐
│     Byte[0]     │ Byte[1] │ Byte[2] │ Byte[3] │
│ 7 6 5 4 3 2 1 0 │ 7 ... 0 │ 7 ... 0 │ 7 ... 0 │
│─────────────────┼─────────┼─────────┼─────────│
│ 1 1 1 1 1 1 1 0 │ red     │ green   │ blue    │
└─────────────────┴─────────┴─────────┴─────────┘
</pre>
#### QOI_OP_RGBA
Same as `QOI_OP_RGB` but we store the full RGBA value, so we need an extra byte for the alpha value. The first byte is `0xff` (or `11111111` in binary).
<pre style="padding:0; display: inline-block;">
┌─ QOI_OP_RGB ────┬─────────┬─────────┬─────────┬─────────┐
│     Byte[0]     │ Byte[1] │ Byte[2] │ Byte[3] │ Byte[4] │
│ 7 6 5 4 3 2 1 0 │ 7 ... 0 │ 7 ... 0 │ 7 ... 0 │ 7 ... 0 │
│─────────────────┼─────────┼─────────┼─────────┼─────────│
│ 1 1 1 1 1 1 1 0 │ red     │ green   │ blue    │ alpha   │
└─────────────────┴─────────┴─────────┴─────────┴─────────┘
</pre>
#### QOI_OP_INDEX
Previously seen pixels are stored in the `index` array at the position given by the following hash function:
```go
func hashPixel(px color.NRGBA) uint8 {
	return (px.R*3 + px.G*5 + px.B*7 + px.A*11) % 64
}

```
Then we store a single byte starting with `00` followed by the index into the array.
<pre style="padding:0; display: inline-block;">
┌─ QOI_OP_INDEX ────┐
│      Byte[0]      │
│ 7 6   5 4 3 2 1 0 │
│─────┼─────────────│
│ 0 0 │    index    │
└─────┴─────────────┘
</pre>
#### QOI_OP_DIFF
<pre style="padding:0; display: inline-block;">
┌─ QOI_OP_DIFF ─────────┐
│        Byte[0]        │
│ 7 6   5 4   3 2   1 0 │
│─────┼─────┼─────┼─────│
│ 0 1 │ dr  │ dg  │ db  │
└─────┴─────┴─────┴─────┘
</pre>
#### QOI_OP_LUMA
<pre style="padding:0; display: inline-block;">
┌─ QOI_OP_LUMA ─────┬───────────────────┐
│      Byte[0]      │      Byte[1]      │
│ 7 6   5 4 3 2 1 0 │ 7 6 5 4   3 2 1 0 │
│─────┼─────────────┼─────────┼─────────│
│ 1 0 │ diff green  │ dr - dg │ db - dg │
└─────┴─────────────┴─────────┴─────────┘
</pre>
#### QOI_OP_RUN
If a pixel is equal to the previously seen pixel we increment the `run` variable by one. Note that because of the way this OP is represented, values `0xfe` and `0xff` are forbidden, since it will collide with the `QOI_OP_RGB` and `QOI_OP_RGBA` representation, so `run` can have a value of at most 63. We save this OP as a single byte starting with `11` followed by the value `run-1`.
<pre style="padding:0; display: inline-block;">
┌─ QOI_OP_RUN ──────┐
│      Byte[0]      │
│ 7 6   5 4 3 2 1 0 │
│─────┼─────────────│
│ 1 1 │   run - 1   │
└─────┴─────────────┘
</pre>

