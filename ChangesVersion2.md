# Changes for version 2.0 #

## Summary ##

Version 2.0 (July 2013) has been a huge change, with drastic rewrites in the implementation,
especially the `PngReader`. The public API has experienced also some changes, not so drastic,
but there are some incompatibilities. This page is oriented towards developers who were
using version 1.x.

## New `PngReader` architecture ##

The class previously used (exclusively) for reading, `PngReader`, was leading towards the "God object"
anti-pattern: it was too big, it did too many things, and it was very difficult to extend or customize.

For example, these things, potentially possible now, were practically impossible before:
  * Extend the `PngReader` to read efficiently the APGN chunks
  * Read PNG variations as MNG
  * Implement PNG with signature
  * Do non-standard thigs with the chunks (including IDAT) eg, reorder them, remove or insert, without parsing the pixels.
  * Read in asynchronous mode, with callbacks (for example, for each read line)
  * Implement the `PngReader`  an an `InputStream` filter, so that it can be place in between call to `ImageIO.read()`  (this is related to the previous)

Furthermore, the tight coupling with the `ImageLine` scanline format used for read/write was not extensible.
An alternate format had been added (bytes instead of integers), but that was inelegant and clumsy.
Some users would like to read to (write from) other formats, perhaps some storage backed by some third party image classes (eg, `BufferedReader` in `java.awt`, or `Bitmap` in Android). The only way was to convert from/to an `ImageLine`, but that was obviously inefficient.

The new architecture allows all this; the price to pay is a bigger class hierarchy (but this is mostly invisible for the typical use cases), and some API incompatibilities (not much). Regarding
performance, both in speed and memory the new version is at least as efficient as the previous, in some cases it performs better.

The main classes of the new architecture are the following
(notice that the most users of the library don't need to understand this, they'll just deal with `PngReader`)

  * `ChunkReader` : short lived object to parse ( no intelligence of its own) a PNG chunk, in one of  three modes (BUFFER, PROCESS, SKIP),         produces a (perhaps empty) `PngChunkRaw`

  * `DeflatedChunkReader` : adds to `ChunkReader`  intelligence to inflate IDAT-like  chunks (deflated stream)

  * A `DeflatedChunksSet` concatenates the ouput of several `DeflatedChunkReader` , and returns the inflated data in fragments of given (perhaps variable) length

  * `IdatSet` (extends `DeflatedChunksSet`) adds IDAT specific intelligence (row unfiltering, PNG sizes, deinterlacing)

  * `ChunkSeqReader` : reads a full sequence of Chunks, including eventally the PNG signature. It contains one active at-a-time `ChunkReader` and, perhaps,  an active `DeflatedChunksSet`). This is quite agnostic, does not have PNG-specific intelligence (it could be used for MGN). It can be used in async (callback) mode, or in sync (polled) mode.

  * `ChunkSeqReaderPNG` : extends  `ChunkSeqReader` with PNG-specific intelligence. It contains a `IdatSet` (extensible), it  has a  `ChunkFactory` to turn the raw parsed chunks into a `PngChunk` list, and checks the PNG chunks order.

  * `PngReader` : wraps a `ChunkSeqReaderPNG` together with a `BufferedStreamFeeder`, to read the bytes from  an input stream  and feed the `ChunkSeqReaderPNG` in sync mode. It allows to read row by row via a `IImageLineSetFactory` which creates a `IImageLineSet` which provides `IImageLine`: the default implementation gives a `ImageLineInt` (backed by a `int[]` scanline). Includes an inmutable `ImageInfo`, and provides high level metadata handling.

  * `PngReaderInt` and `PngReaderByte` are trivial (especially the first) extensions of `PngReader`, that use `ImageLineInt` and `ImageLineByte`


## Most visible (most incompatible!) changes ##

  * `PngFileHelper` is dead (and nothing of value was lost), just use good old plain constructors to instantiate `PngReader` and `PngWriter`

  * In `PngWriter` the pair `copyChunksFirst()` ,`copyChunksLast()` is also gone, now just call once  `copyChunksFrom()`, at the beginning. The behaviour is not the same - but it's for better.

Before version 2.0:
```
PngReader pngr = FileHelper.createPngReader(file);
PngWriter png1 = FileHelper.createPngWriter(file2,imginfo,overwrite);
pngw.copyChunksFirst(pngr, copyPolicy);
///... read and copy pixels
pngw.copyChunksLast(pngr, copyPolicy);
pngr.end();
pngw.end();
```

Version 2.0:
```
PngReader pngr = new PngReader(file);
PngWriter pngw = new PngWriter(file2,imginfo,overwrite);
png1.copyChunksFrom(pngr.getChunksList());
///... read and copy pixels
pngr.end(); // Warning: order can be important, close first the reader.
pngw.end();
```

  * `ImageLine` (and reading the row as a plain int array) is gone. `PngReader.readRow()` returns a `IImageLine`, which is a customizable interface that knows how to store pixels in its own format.

As a convenience, two implementations are provided: `ImageLineInt` and `ImageLineByte`,
and `PngReader.readRow()` by default returns a `ImageLineInt`  (you can change this, to a
`ImageLineByte` or to other format by subclassing, or by setting an alternate `ImageLineSet` factory).
Two further unimportant but handy facilities: a `PngReader.readRowInt()` does the casting for you,
and two trivial sublcasses `PngReaderInt`, `PngReaderByte` are also provided.

  * The rows must be read in ascending order (as always), but to add the row number is now not required nor recommended. It's understood that `readRow()` means "read next row", and the caller know what he is doing.

Before version 2.0:
```
     for (int row = 0; row < pngr.imgInfo.rows; row++) {
                ImageLine line= pngr.readRow(row);
                line.scanline[....] ;//do something
		pngw.writeRow(line, row);
     }
```

Version 2.0:
```
     // also : while(pngr.hasMoreRows())
     for (int row = 0; row < pngr.imgInfo.rows; row++) { 
                ImageLineInt line= (ImageLineInt)pngr.readRow();
                line.getScanline()[....] ;//do something
		pngw.writeRow(line);
     }
```


---


_If you find information missing or have some observation about version 2.0, please comment below_