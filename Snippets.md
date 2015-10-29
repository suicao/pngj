# Snippets and recipes #

_WARNING: Some snippets are still not adapted/optimized/tested for version 2.0_


Since 2.1.0 the test source (not included in the main jar) has a package
[ar.com.hjg.pngj.cli](https://code.google.com/p/pngj/source/browse/#git%2Fsrc%2Ftest%2Fjava%2Far%2Fcom%2Fhjg%2Fpngj%2Fcli) which includes some command line utilities based on PNG.



## Flip horizontally an image ##

This is basically the code for `SampleMirrorImage` class in the samples. Notice that this works for every image model - that's because even for the "packed" formats (low bitdepth: one byte packs several samples of 1-2-4 bits) we always have (if we use the standard `ImageLineInt` storage) one sample per array element.

```

void static mirrorPng(String orig,String dest) {
 PngReader pngr = new PngReader(orig);
 PngWriter pngw = new PngWriter(dest, pngr.imgInfo, false);
 pngw.copyChunksFrom(pngr.getChunksList());
 for (int row = 0; row < pngr.imgInfo.rows; row++) {
	ImageLineInt line = (ImageLineInt)pngr.readRow();
	mirrorLine(line, pngr.imgInfo);
	pngw.writeRow(line);
 }
 pngr.end();
 pngw.end();
}

void static mirrorLine(ImageLineInt line, ImageInfo iminfo) {
	int channels = iminfo.channels;
	int[] imlinei = imline.getScanline();
	for (int c1 = 0, c2 = iminfo.cols - 1; c1 < c2; c1++, c2--) {
		for (int i = 0; i < channels; i++) {
			int s1 = c1 * channels + i; // sample left
			int s2 = c2 * channels + i; // sample right
			int aux = imlinei[s1];
			imlinei[s1] = imlinei[s2];
		}

        }
}

```

Two notes about memory use:
  * The example uses `ImageLineInt` which wraps an integer array with `cols x channels` elements; the memory usage is then about `cols x channels x 4`. By using `PngReaderByte` we can load each line in an `ImageLineByte`, reducing the memory by a factor of 4 - but we'd loose resolution if image has 16bits depth.
  * The above is not true if the image is interlaced (not frequent, and not recommended), in which case the full image must actually be loaded. This is transparent to the programmer, except that the memory usage multiplies by `rows`. This is practically unavoidable.  Of course, we call always call `if(pngr.isInterlaced())` to detect and disallow interlaced images if we wish to do so.

## Read all textual chunks ##

This loads all chunks, skipping pixels data, and process the textual chunks. Bear in mind that there are [three types of text chunks](http://www.libpng.org/pub/png/spec/1.2/PNG-Chunks.html#C.Anc-text), and `iTxt` includes some extra info. Also, remember that textual chunks with repeated keys are allowed.

```
  PngReader pngr = new PngReader(file);
  pngr.readSkippingAllRows(); // reads only metadata
  for (PngChunk c : pngr.getChunksList().getChunks()) {
      if (!ChunkHelper.isText(c))   continue;
      PngChunkTextVar ct = (PngChunkTextVar) c;
      String key = ct.getKey();
      String val = ct.getVal();
      // ... 
  }
  pngr.end(); // not necessary here, but good practice
```



## Use a `BufferedImage` with `PngWriter` ##

_TODO: in version 2.0 this could be done much more efficiently. PENDING_

PNGJ is, by design, decoupled from `java.awt.*`. If you want to write or read from a `BufferedImage`, you must adapt the format. For example, the following code writes a `BufferedImage` of type `TYPE_INT_ARGB` to a RGBA8 PNG image:

```
	/**
	 * 
	 * @param bi BufferedImage of TYPE_INT_ARGB or TYPE_INT_RGB
	 * @param os
	 */
	public static void writeARGB(BufferedImage bi, OutputStream os) {
		if(bi.getType() != BufferedImage.TYPE_INT_ARGB) throw new PngjException("This method expects  BufferedImage.TYPE_INT_ARGB" );
		ImageInfo imi = new ImageInfo(bi.getWidth(), bi.getHeight(), 8, true);
		PngWriter pngw = new PngWriter(os, imi);
		// pngw.setCompLevel(6); // tuning
		// pngw.setFilterType(FilterType.FILTER_PAETH); // tuning
		DataBufferInt db =((DataBufferInt) bi.getRaster().getDataBuffer());
		if(db.getNumBanks()!=1) throw new PngjException("This method expects one bank");
		SinglePixelPackedSampleModel samplemodel =  (SinglePixelPackedSampleModel) bi.getSampleModel();
		ImageLine line = new ImageLine(imi);
		int[] dbbuf = db.getData();
		for (int row = 0; row < imi.rows; row++) {
			int elem=samplemodel.getOffset(0,row);
			for (int col = 0,j=0; col < imi.cols; col++) {
				int sample = dbbuf[elem++];
				line.scanline[j++] =  (sample & 0xFF0000)>>16; // R
				line.scanline[j++] =  (sample & 0xFF00)>>8; // G
				line.scanline[j++] =  (sample & 0xFF); // B
				line.scanline[j++] =  (((sample & 0xFF000000)>>24)&0xFF); // A
			}
			pngw.writeRow(line, row);
		}
		pngw.end();
	}
```

To write a `TYPE_4BYTE_ABGR` image (again, to PNGA8 ), you'd change the data buffer access, eg:

```
                ...
		DataBufferByte db =((DataBufferByte) bi.getRaster().getDataBuffer());
		if(db.getNumBanks()!=1) throw new PngjException("This method expects one bank");
		ComponentSampleModel samplemodel =  (ComponentSampleModel) bi.getSampleModel();
		ImageLine line = new ImageLine(imi);
		byte[] dbbuf = db.getData();
		for (int row = 0; row < imi.rows; row++) {
			int elem=samplemodel.getOffset(0,row);
			for (int col = 0,j=0; col < imi.cols; col++,elem+=7) {
				line.scanline[j++] =  dbbuf[elem--]; // R
				line.scanline[j++] =  dbbuf[elem--]; // G
				line.scanline[j++] =  dbbuf[elem--]; // B
				line.scanline[j++] =  dbbuf[elem]; //A
			}
			pngw.writeRow(line, row);
		}
                ...
```

## Image tiling (identical sizes and color models) ##
```
/**
 * Takes several tiles and join them in a single image
 * 
 * @param tiles            Filenames of PNG files to tile
 * @param dest            Destination PNG filename
 * @param nTilesX            How many tiles per row?
 */
public class SampleTileImage {

	public static void doTiling(String tiles[], String dest, int nTilesX) {
		int ntiles = tiles.length;
		int nTilesY = (ntiles + nTilesX - 1) / nTilesX; // integer ceil
		ImageInfo imi1, imi2; // 1:small tile   2:big image
		PngReader pngr = new PngReader(new File(tiles[0]));
		imi1 = pngr.imgInfo;
		PngReader[] readers = new PngReader[nTilesX];
		imi2 = new ImageInfo(imi1.cols * nTilesX, imi1.rows * nTilesY, imi1.bitDepth, imi1.alpha, imi1.greyscale,
				imi1.indexed);
		PngWriter pngw = new PngWriter(new File(dest), imi2, true);
		// copy palette and transparency if necessary (more chunks?)
		pngw.copyChunksFrom(pngr.getChunksList(), ChunkCopyBehaviour.COPY_PALETTE
				| ChunkCopyBehaviour.COPY_TRANSPARENCY);
 	        pngr.readSkippingAllRows(); // reads only metadata	       
                pngr.end(); // close, we'll reopen it again soon
		ImageLineInt line2 = new ImageLineInt(imi2);
		int row2 = 0;
		for (int ty = 0; ty < nTilesY; ty++) {
			int nTilesXcur = ty < nTilesY - 1 ? nTilesX : ntiles - (nTilesY - 1) * nTilesX;
			Arrays.fill(line2.getScanline(), 0);
			for (int tx = 0; tx < nTilesXcur; tx++) { // open serveral readers
				readers[tx] = new PngReader(new File(tiles[tx + ty * nTilesX]));
				readers[tx].setChunkLoadBehaviour(ChunkLoadBehaviour.LOAD_CHUNK_NEVER);
				if (!readers[tx].imgInfo.equals(imi1))
					throw new RuntimeException("different tile ? " + readers[tx].imgInfo);
			}
			for (int row1 = 0; row1 < imi1.rows; row1++, row2++) {
				for (int tx = 0; tx < nTilesXcur; tx++) {
					ImageLineInt line1 = (ImageLineInt) readers[tx].readRow(row1); // read line
					System.arraycopy(line1.getScanline(), 0, line2.getScanline(), line1.getScanline().length * tx,
							line1.getScanline().length);
				}
				pngw.writeRow(line2, row2); // write to full image
			}
			for (int tx = 0; tx < nTilesXcur; tx++)
				readers[tx].end(); // close readers
		}
		pngw.end(); // close writer
	}

	public static void main(String[] args) {
		doTiling(new String[] { "t1.png", "t2.png", "t3.png", "t4.png", "t5.png", "t6.png" }, "tiled.png", 2);
		System.out.println("done");
	}
}
```
## Extract frames from APNG animated image ##


---

_TODO This must me updated to version 2.0, this shoud be doable much more cleanly and efficiently now._


---


PNGJ does not support APNG, actually it dislikes it: by default it does not even load APGN chunks. Anyway, here is a dirty snippet that extract the frames from a APGN - this is not very elegant, nor totally robust (it does not work with partial frames, does not understand overlays, etc) but for simple images it works.

```
public class ApngSplit {

	private static final String PREFIX = "apngf";

	/** reads a APNG file and tries to split it into its frames */
	public static void process(File orig) throws Exception {
		PngReader pngr = FileHelper.createPngReader(orig);
		File dest = new File(orig.getParent(), PREFIX + "0_" + orig.getName());
		PngWriter pngw = FileHelper.createPngWriter(dest, pngr.imgInfo, true);
		System.out.println("writing default frame " + pngw.getFilename());
		pngr.setChunkLoadBehaviour(ChunkLoadBehaviour.LOAD_CHUNK_ALWAYS);
		pngr.setMaxBytesMetadata(Integer.MAX_VALUE);
		pngr.setSkipChunkIds(new String[] {}); // we don't want to skip APNG chunks here
		int copyPolicy = ChunkCopyBehaviour.COPY_PALETTE | ChunkCopyBehaviour.COPY_ALL_SAFE;
		pngw.copyChunksFirst(pngr, copyPolicy);
		for (int row = 0; row < pngr.imgInfo.rows; row++) 
			pngw.writeRow(pngr.readRow(row), row);
		pngr.end();
		pngw.copyChunksLast(pngr, copyPolicy);
		pngw.end();
		processExtra(orig, pngr.getChunksList());
	}

	/** writes each APNG extra frame in a single PNG file */
	private static void processExtra(File orig, ChunksList chunks) throws Exception {
		int numframe = 0;
		FileOutputStream os = null;
		boolean afterIdat = false;
		for (PngChunk chunkApng : chunks.getChunks()) {
			if (chunkApng.id.equals("IDAT"))
				afterIdat = true;
			if (chunkApng.id.equals("fcTL") && afterIdat) {
				numframe++;
				if (os != null)
					endPng(chunks, os);
				File dest = new File(orig.getParent(), PREFIX + numframe + "_" + orig.getName());
				System.out.println("writing frame " + numframe + " : " + dest);
				os = new FileOutputStream(dest);
				beginPng(chunks, os);
			}
			if (chunkApng.id.equals("fdAT")) {
				ChunkRaw crawf = chunkApng.createRawChunk();
				int seq = PngHelperInternal.readInt4fromBytes(crawf.data, 0);
				ChunkRaw crawi = new ChunkRaw(crawf.len - 4, ChunkHelper.b_IDAT, true);
				System.arraycopy(crawf.data, 4, crawi.data, 0, crawi.data.length);
				crawi.writeChunk(os);
			}
		}
		if (os != null)
			endPng(chunks, os);
	}

	private static void endPng(ChunksList chunks, FileOutputStream fos) throws Exception {
		chunks.getById1(PngChunkIEND.ID).createRawChunk().writeChunk(fos);
		fos.close();
	}

	private static void beginPng(ChunksList chunks, FileOutputStream fos) throws Exception {
		fos.write(new byte[] { -119, 80, 78, 71, 13, 10, 26, 10 }); // signature
		chunks.getById1(PngChunkIHDR.ID).createRawChunk().writeChunk(fos);
		// try to write palette, if present - we could add other chunks
		PngChunk plte = chunks.getById1(PngChunkPLTE.ID);
		if (plte != null)
			plte.createRawChunk().writeChunk(fos);
	}

	public static void main(String[] args) throws Exception {
		process(new File("C:/temp/029.png"));
	}

}

```