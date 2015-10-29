<h1> PNGJ: a PNG Java library</h1>

PNGJ is a library to read and write **PNG** images.
It's fully written in Java, without any dependencies on  <tt>javax.imageio</tt>, AWT or any third party libraries.

It provides a simple API for progressive (sequential line-oriented) reading and writing.
It's specially suitable for huge images, which one does not want to  load fully in memory, eg. (as a BufferedImage).

It supports (since June 2011) **all PNG spec color models and bitdepths**:

> True color: 8-16 bpp, with and without alpha channel (RGB/RGBA)

> Grayscale:  1-2-4-8-16 bpp, with and without alpha channel

> Indexed: palette with 1-2-4-8 bits

It support interlaced images reading (since v 1.1).

It has full **metadata** support: it reads and writes all the standard [ancillary Chunks](http://www.w3.org/TR/PNG/#11Ancillary-chunks), and it allows to register additional custom chunks.

It has been tested (read and write) against the [full standard PNG test suite](http://www.schaik.com/pngsuite/)

A quick example of use: the sample code below reads a PNG image file (true colour RGB/RGBA, 8 or 16 bpp), decreases the red channel value by 50% and writes it into another PNG file.

```

	public static void convert(String origFilename, String destFilename) {
		PngReader pngr = new PngReader(new File(origFilename));
		System.out.println(pngr.toString());
		int channels = pngr.imgInfo.channels;
		if (channels < 3 || pngr.imgInfo.bitDepth != 8)
			throw new RuntimeException("This method is for RGB8/RGBA8 images");
		PngWriter pngw = new PngWriter(new File(destFilename), pngr.imgInfo, true);
		pngw.copyChunksFrom(pngr.getChunksList(), ChunkCopyBehaviour.COPY_ALL_SAFE);
		pngw.getMetadata().setText(PngChunkTextVar.KEY_Description, "Decreased red and increased green");
		for (int row = 0; row < pngr.imgInfo.rows; row++) { // also: while(pngr.hasMoreRows()) 
			IImageLine l1 = pngr.readRow();
			int[] scanline = ((ImageLineInt) l1).getScanline(); // to save typing
			for (int j = 0; j < pngr.imgInfo.cols; j++) {
				scanline[j * channels] /= 2;
				scanline[j * channels + 1] = ImageLineHelper.clampTo_0_255(scanline[j * channels + 1] + 20);
			}
			pngw.writeRow(l1);
		}
		pngr.end(); // it's recommended to end the reader first, in case there are trailing chunks to read
		pngw.end();
	}
```


The following sample code generates an RGB8 (orange-ish gradient) image to an OutputStream. Because only an image line is allocated, this would allow the creation of very large images with low memory usage.
This example also shows some metadata handling.

```
		ImageInfo imi = new ImageInfo(cols, rows, 8, false); // 8 bits per channel, no alpha
		// open image for writing to a output stream
		PngWriter png = new PngWriter(outputStream, imi);
		// add some optional metadata (chunks)
		png.getMetadata().setDpi(100.0);
		png.getMetadata().setTimeNow(0); // 0 seconds fron now = now
		png.getMetadata().setText(PngChunkTextVar.KEY_Title, "just a text image");
		png.getMetadata().setText("my key", "my text");
		ImageLineInt iline = new ImageLineInt(imi);
		for (int col = 0; col < imi.cols; col++) { // this line will be written to all rows
			int r = 255;
			int g = 127;
			int b = 255 * col / imi.cols;
			ImageLineHelper.setPixelRGB8(iline, col, r, g, b); // orange-ish gradient
		}
		for (int row = 0; row < png.imgInfo.rows; row++) {
			png.writeRow(iline);
		}
		png.end();
	}
```

See more [Snippets](Snippets.md), also in the
samples included in the [test package](https://code.google.com/p/pngj/source/browse/#git%2Fsrc%2Ftest%2Fjava%2Far%2Fcom%2Fhjg%2Fpngj%2Fsamples), classes starting with `Sample*`, and also the unit tests.

Any questions? Read the [FAQ](FAQ.md) or the [Javadocs](https://pngj.googlecode.com/git/site/apidocs/index.html)