# PNGJ #

Pure Java library for efficient decoding/encoding of PNG images

Downloads: You can download the latest release from here http://hjg.com.ar/pngj/ or use the [Maven Central repository](http://search.maven.org/#browse%7C-1552047720)

---

THIS PROJECT IS BEING MIGRATED TO GITHUB:

> https://github.com/leonbloy/pngj/

---

### Main features ###

  * Very efficient in memory usage and speed.
  * Pure Java (requires 5 or greater).
  * Small and self contained. No dependencies on third party libraries, nor even on `java.awt.* /javax.imageio.*`
  * Runs on GAE (Google App Engine) and Android
  * Allows to read/write progressively, row by row. This means that you can process huge images without needing to load them fully in memory.
  * Reads and writes all PNG color models (true color, gray, palette, all bit depths: 1-2-4-8-16, with-without alpha). Interlaced PNG is supported (though not welcomed) for reading.
  * Full support for metadata ("chunks") handling.
  * The format of the pixel data (read and write) is extensible (since 2.0) and very efficient (no double copies).
  * Supports (since 2.0) asyncronous reading and low level tweaking and extension in the reader.
  * Open source (Apache licence). Available in Maven Central repository.

Note that this is a relatively low-level library. It does not provide any high-level image processing (eg, resizing, colour conversions), nor tries to abstract the concrete image format (as `BufferedImage` does, for example). In particular, the default format of the scanlines (as wrapped in `ImageLineInt` or `ImageLineByte`) is not abstract, the meaning of the values depends on the image color model and bitdepth.

### More info ###

  * [Overwiew](http://code.google.com/p/pngj/wiki/Overview)
  * [FAQ](http://code.google.com/p/pngj/wiki/FAQ)
  * [Javadocs](https://pngj.googlecode.com/git/site/apidocs/index.html)
  * [History](https://code.google.com/p/pngj/source/browse/src/site/RELEASE-NOTES.txt)
  * [Who's using it](http://code.google.com/p/pngj/wiki/Users)


---


C# port: http://code.google.com/p/pngcs


---


Questions, comments? [Comment here](https://groups.google.com/forum/#!forum/pngj-discuss)

To keep updated of new releases and news: [feeds](https://code.google.com/p/pngj/feeds)