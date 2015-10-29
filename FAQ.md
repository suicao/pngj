# FAQ #

  * [Does this library support all PNG formats?](#Does_this_library_support_all_PNG_formats?.md)

  * [Why would I prefer this to other image libraries?](#Why_would_I_prefer_this_to_other_image_libraries?.md)

  * [Why there are several jars?](#Why_there_are_several_jars?.md)

  * [How do I read/write the pixels values?](#How_do_I_read/write_the_pixels_values?.md)

  * [What about image metadata?](#What_about_image_metadata?.md)

  * [What are the Java requirements?](#What_are_the_Java_requirements?.md)

  * [Can it be deployed in GAE (Google App Engine) sandbox?](#Can_it_be_deployed_in_GAE_sandbox?.md)

  * [Only Java is supported?](#Only_Java_is_supported?.md)

## Does this library support all PNG formats? ##

Short answer: yes<s>, excepted for interlaced.</s>

Long answer: Since version 1.x, all valid PNG images can be read.
All color modes are fully implemented: true color (RGB), indexed (paletted), grayscale (black/white), with any bit depth (1-2-4-8-16) and with optional alpha channel. Interlaced images can be read (though in that case the row-by-row read is not efficient), but they are writen as non-interlaced. All standard internal filters/compression are implemented. All standard ancillary chunks are understood, and can be read/copied/created.
The [SampleMirrorImage sample class](https://code.google.com/p/pngj/source/browse/src/test/java/ar/com/hjg/pngj/samples/SampleMirrorImage.java) (an example that reads an arbitrary PNG  images, mirrors each row, and writes it back) has been tested to work with the entire [PNG test-suite](http://www.schaik.com/pngsuite/).

## Why would I prefer this to other image libraries? ##

There are several other image libraries. Java SE provides the `javax.imageio` package which, together with the `java.awt.image` package (`BufferedImage`) allows to read-write images in many formats. In typical scenarios, that's the simplest and recommended way to deal with images in Java.

On the other hand, PNGJ is relatively low-level, and is tightly coupled with the PNG spec: it does not try to abstract from the image format and color models, nor does it provide high-level general image manipulation capabilities (eg. image transformations, color adjustments, etc). This can be a limitation, but it also have some advantages:

  * I/O is quite efficient, in speed and specially in **memory usage**. Because it reads/writes in sequentially (as a stream, row by row), it does not require the full image to be in memory (except for interlaced images)- just the current line.

  * Related to the above, it can deal with **huge images**. Both in pixel dimensions (say, a 80000 x 80000 RGB image) and in total size (say a PNG files greater than 2GB). There are practically no limits.

  * It read/writes all the PNG color models, including **16 bits per channel** images (16/32/48/64 bits per pixel).

  * It gives the programmer much flexibility and it does not make guesses. For example, it can store samples  in arrays of integers or in bytes, but the decision is made by the client code (the programmer), not by the library.

  * The library is small and self contained: it has **no dependencies** on third party libraries - nor even the `java.awt.*` package. **It works in [GAE](https://developers.google.com/appengine/) and Android**.

  * It handles standard PNG **metadata** (chunks), and allows to register additional custom chunks.

## Why there are several jars? ##

The **standard jar** (`pngj.jar`) includes all the runtime (binary classes). This is all you normally need to use the library in your project.

The **source jar** (`pngj-src.jar`) includes the source code

The Javadocs are inside the archive (tgz or zip).

## How do I read/write the pixels values? ##

Each `PngReader`'s `readRow()` call returns a `IImageLine`, which represents a PNG image row; the concrete implementation is extensible. But the default included implementation (`PngReader` or `PngReaderInt`) returns a [ImageLineInt](https://code.google.com/p/pngj/source/browse/src/main/java/ar/com/hjg/pngj/ImageLineInt.java). This class wraps an scaline (`getScanline()`) which is an `int[]`  array. Each **element of this array correspond to a image sample** ; so, the array length is (at least) `columns x channels`.

An alternative format is [ImageLineByte](https://code.google.com/p/pngj/source/browse/src/main/java/ar/com/hjg/pngj/ImageLineByte.java), which is identical except that it stores each sample in a byte (advantage: less memory usage, optimal speed for 8bits images; disadvantages: loses resolution if image is 16bits-per-channel, and it's more cumbersome to do arithmetic -bytes in Java are signed).

The layout of sample values inside the scanline is as follows:

For true colour images, RGB/RGBA the samples are in RGB(A) order: `R G B R G B ... ` (no alpha) or `R G B A R G B A ...` (with alpha); the values are not scaled: they will be in the 0-255 range only if bitdepth=8 (0-65535 if bitdepth=16, 0-15 if bitdepth=4, etc).
For indexed images, each sample value correspond to the palette index `I I ...`. For grayscale G/GA images it's the gray value `G G G ...` (no alpha) or `G A G A ...` (with alpha).

## What about image metadata? ##

The API provides full low level access to the PNG "[chunks](http://www.w3.org/TR/PNG/#4Concepts.FormatChunks)", which provides some image metadata handling.

As version 0.71 (March 2012), full support is provided for ALL the standard chunks **PLTE** (Palette), **pHYS** (physical resolution),  **tRNS** (transparency),  **gAMA** (Gamma), **cHRM** (chroma), **iCCP** (ICC Profile), **sBIT** (significant bits), **sRGB** (standard RGB colour space), **tEXt** (arbitrary text, in `key=value` form, Latin1 encoding), **zTXt** (idem, with compressed text), **iTXt** (idem, with localized tagname and text in UTF-8), **bKGD** (background color), **tIME** (creation time), **hIST** (histogram), **sPLT** (suggested palette). Extensions added in 0.91: **oFFs** (offset) and **sTER** (stereoscopic). It's also possible to register additional chunk types from client code, by extending the `ChunkFactory`, see an example in [CustomChunkTest](https://code.google.com/p/pngj/source/browse/src/test/java/ar/com/hjg/pngj/test/CustomChunkTest.java)

The API also provides some methods to do a general transfer from the chunks of a source image to a new target, with various levels of detail.
Relevant methods to look at: `PngReader.setChunkLoadBehaviour()` and `PngWriter.copyChunksFrom()`. See also samples, eg
[CopyChunksTest](https://code.google.com/p/pngj/source/browse/src/test/java/ar/com/hjg/pngj/CopyChunksTest.java).

Another examples: [SampleRemoveGama](https://code.google.com/p/pngj/source/browse/src/test/java/ar/com/hjg/pngj/samples/SampleRemoveGama.java) shows how to get the GAMA chunk and remove it. [SampleCreateOrangeGradient](https://code.google.com/p/pngj/source/browse/src/test/java/ar/com/hjg/pngj/samples/SampleCreateOrangeGradient.java) adds several chunks to a new image: physical resolution, textual information, creation time.

Notice that the "Colour space information" chunks (cHRM,iCCP,sBIT, etc) can be read and writen but no translation is done at the pixel level when reading or writing an image line: the row of pixels that sees this library (eg `ImageLineInt`) always correspond to the raw values stored in the image data. If you want, for example, to read the pixel values after some colour correction is applied according to those chunks, you must do it yourself.

## What are the Java requirements? ##

Java 5.0 is required. No extra dependencies are needed.

## Can it be deployed in GAE (Google App Engine) sandbox? ##

Yes. The classes used in pnjg.jar are in GAE's white list. The only
exception is, naturally, that you cannot write to a file, so `new PngWriter(File imagefile)` will fail, you must use `new PngWriter(OutputStream stream)`

## Only Java is supported? ##

This is a Java library. I've done a .NET port, in C#, available as other project: [PNGCS](http://code.google.com/p/pngcs), it covers the same functionality, though it's a bit less efficient (mainly because common Java JRE include native Zlib routines).