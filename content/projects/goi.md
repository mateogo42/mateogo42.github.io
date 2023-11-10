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
