package de.tuberlin.dima.minidb.io.tables;

import java.io.EOFException;
import java.io.File;
import java.io.IOException;
import java.io.RandomAccessFile;
import java.nio.ByteBuffer;
import java.nio.ByteOrder;
import java.nio.channels.FileChannel;
import java.nio.channels.FileLock;
import java.nio.channels.OverlappingFileLockException;

import de.tuberlin.dima.minidb.Constants;
import de.tuberlin.dima.minidb.api.AbstractExtensionFactory;
import de.tuberlin.dima.minidb.catalogue.ColumnSchema;
import de.tuberlin.dima.minidb.catalogue.TableSchema;
import de.tuberlin.dima.minidb.core.BasicType;
import de.tuberlin.dima.minidb.core.DataType;
import de.tuberlin.dima.minidb.io.cache.CacheableData;
import de.tuberlin.dima.minidb.io.cache.PageFormatException;
import de.tuberlin.dima.minidb.io.cache.PageSize;
import de.tuberlin.dima.minidb.io.manager.ResourceManager;


public class TableResourceManager extends ResourceManager {

	private static final int TABLE_HEADER_MAGIC_NUMBER = 0xDEAFD00D;


	private static final int TABLE_HEADER_COLUMN_ATTRIBUTE_NULLABLE_MASK = 0x1;

	private static final int TABLE_HEADER_COLUMN_ATTRIBUTE_UNIQUE_MASK = 0x2;


	private static final AbstractExtensionFactory pageFactory = AbstractExtensionFactory.getExtensionFactory();


	private final FileChannel ioChannel;


	private final FileLock theLock;


	private final TableSchema schema;


	private final int pageSize;


	private final int firstDataPageNumber;


	private int lastDataPageNumber;



	protected TableResourceManager(RandomAccessFile fileHandle) throws IOException, PageFormatException {

		try {
			this.ioChannel = fileHandle.getChannel();
			try {
				this.theLock = this.ioChannel.tryLock();
			} catch (OverlappingFileLockException oflex) {
				throw new IOException("Table file locked by other consumer.");
			}

			if (this.theLock == null) {
				throw new IOException("Could acquire table file handle for exclusive usage. File locked otherwise.");
			}
		} catch (Throwable t) {

			makeBestEffortToClose();


			if (t instanceof IOException) {
				throw (IOException) t;
			} else {
				throw new IOException("An error occured while opening the table: " + t.getMessage());
			}
		}

		this.ioChannel.position(0);

		this.schema = readTableHeader(this.ioChannel);
		this.pageSize = this.schema.getPageSize().getNumberOfBytes();


		this.firstDataPageNumber = (int) (this.ioChannel.position() / this.schema.getPageSize().getNumberOfBytes()) + 1;
		this.lastDataPageNumber = (int) ((this.ioChannel.size() - 1) / this.schema.getPageSize().getNumberOfBytes());
	}


	protected TableResourceManager(RandomAccessFile fileHandle, TableSchema schema) throws IOException {

		try {
			this.ioChannel = fileHandle.getChannel();
			try {
				this.theLock = this.ioChannel.tryLock();
			} catch (OverlappingFileLockException oflex) {
				throw new IOException("Table file locked by other consumer.");
			}

			if (this.theLock == null) {
				throw new IOException("Could acquire table file handle for exclusive usage. File locked otherwise.");
			}
		} catch (Throwable t) {

			makeBestEffortToClose();


			if (t instanceof IOException) {
				throw (IOException) t;
			} else {
				throw new IOException("An error occured while opening the table: " + t.getMessage());
			}
		}


		this.schema = schema;
		this.pageSize = schema.getPageSize().getNumberOfBytes();


		this.ioChannel.position(0);
		writeTableHeader(schema, this.ioChannel);


		this.firstDataPageNumber = (int) (this.ioChannel.position() / schema.getPageSize().getNumberOfBytes()) + 1;
		this.lastDataPageNumber = this.firstDataPageNumber - 1;
	}


	@Override
	public synchronized void closeResource() throws IOException {
		try {
			this.theLock.release();
			this.ioChannel.close();
		} catch (Throwable t) {

			makeBestEffortToClose();


			if (t instanceof IOException) {
				throw (IOException) t;
			} else {
				throw new IOException("An error occured while opening the table: " + t.getMessage());
			}
		}
	}


	private void makeBestEffortToClose() {

		if (this.theLock != null) {
			try {
				this.theLock.release();
			} catch (Throwable ignored) {

			}
		}

		if (this.ioChannel != null) {
			try {
				this.ioChannel.close();
			} catch (Throwable ignored) {

			}
		}
	}


	public TableSchema getSchema() {
		return this.schema;
	}


	@Override
	public PageSize getPageSize() {
		return this.schema.getPageSize();
	}
	public int getFirstDataPageNumber() {
		return this.firstDataPageNumber;
	}


	public int getLastDataPageNumber() {
		return this.lastDataPageNumber;
	}



	@Override
	public synchronized void truncate() throws IOException {
		this.ioChannel.truncate(this.firstDataPageNumber * this.schema.getPageSize().getNumberOfBytes());
		this.lastDataPageNumber = this.firstDataPageNumber - 1;
	}


	@Override
	public final synchronized TablePage reserveNewPage(byte[] buffer) throws PageFormatException {

		if (buffer.length < this.schema.getPageSize().getNumberOfBytes()) {
			throw new IllegalArgumentException("The buffer to initialize the page to is too small.");
		}


		int nextEmptyPageNumber = this.lastDataPageNumber >= this.firstDataPageNumber ? this.lastDataPageNumber + 1 : this.firstDataPageNumber;

		TablePage newPage = pageFactory.initTablePage(this.schema, buffer, nextEmptyPageNumber);


		this.lastDataPageNumber = nextEmptyPageNumber;

		return newPage;
	}


	@Override
	public final synchronized TablePage reserveNewPage(byte[] buffer, Enum<?> type) throws PageFormatException {
		return reserveNewPage(buffer);
	}


	@Override
	public void writePageToResource(byte[] buffer, CacheableData wrapper) throws IOException {
		int pageNumber = wrapper.getPageNumber();

		if (Constants.DEBUG_CHECK) {

			if (pageNumber < this.firstDataPageNumber) {
				throw new IOException("Page number " + pageNumber + " is not valid. " + "First data page is " + this.firstDataPageNumber + ".");
			}

			if (buffer.length != this.pageSize) {
				throw new IllegalArgumentException("The given buffer does not match for the specified page size (" + this.pageSize + " bytes).");
			}
		}

		ByteBuffer b = ByteBuffer.wrap(buffer, 0, this.pageSize);


		try {
			long position = (this.pageSize * (long) pageNumber);
			writeBuffer(this.ioChannel, b, position);
		} catch (IOException ioex) {
			throw new IOException("Page " + pageNumber + " could not be written to the table file.", ioex);
		}
	}


	@Override
	public void writePagesToResource(byte[][] buffers, CacheableData[] wrappers) throws IOException {
		if (Constants.DEBUG_CHECK) {

			if (buffers.length != wrappers.length) {
				throw new IllegalArgumentException("Unequal number of buffers and wrappers provided.");
			}


			if (buffers.length <= 0) {
				throw new IllegalArgumentException("At least one buffer should be provided.");
			}
		}

		int pageNumber = wrappers[0].getPageNumber();

		if (Constants.DEBUG_CHECK) {

			if (pageNumber < this.firstDataPageNumber) {
				throw new IOException("Page number " + pageNumber + " is not valid. " + "First data page is " + this.firstDataPageNumber + ".");
			}

			for (int i = 0; i < buffers.length; i++) {

				if (wrappers[i].getPageNumber() != pageNumber + i) {
					throw new IOException("Page number " + wrappers[i].getPageNumber() + " of page at position " + i + " is not sequential.");
				}


				if (buffers[i].length != this.pageSize) {
					throw new IllegalArgumentException("The given buffer does not match for the specified page size (" + this.pageSize + " bytes).");
				}
			}
		}

		ByteBuffer[] b = new ByteBuffer[buffers.length];
		for (int i = 0; i < buffers.length; i++) {
			b[i] = ByteBuffer.wrap(buffers[i], 0, this.pageSize);
		}


		try {
			this.ioChannel.position(this.pageSize * (long) pageNumber);
			long totalSize = buffers.length * this.pageSize;
			long bytesRemaining = buffers.length * this.pageSize;
			int currFirstBuffer = 0;
			do {



				bytesRemaining -= this.ioChannel.write(b, currFirstBuffer, buffers.length - currFirstBuffer);
				currFirstBuffer = (int) ((totalSize - bytesRemaining) / this.pageSize);
			} while (bytesRemaining > 0);
		} catch (IOException ioex) {
			throw new IOException("Page sequence [" + pageNumber + ", " + (pageNumber + buffers.length - 1) + "] could not be written to the table file.", ioex);
		}
	}


	@Override
	public TablePage readPageFromResource(byte[] buffer, int pageNumber) throws IOException {
		if (Constants.DEBUG_CHECK) {

			if (pageNumber < this.firstDataPageNumber) {
				throw new IOException("Page number " + pageNumber + " is not valid. " + "First data page is " + this.firstDataPageNumber + ".");
			}


			if (buffer.length != this.pageSize) {
				throw new IOException("Buffer is not big enough to hold a page.");
			}
		}


		ByteBuffer b = ByteBuffer.wrap(buffer, 0, this.pageSize);

		try {
			long position = (this.pageSize * (long) pageNumber);
			readIntoBuffer(this.ioChannel, b, position, this.pageSize);
		} catch (IOException ioex) {
			throw new IOException("Page " + pageNumber + " could not be read from table file.", ioex);
		}


		try {
			return pageFactory.createTablePage(this.schema, buffer);
		} catch (PageFormatException pfex) {
			throw new IOException("Page could not be fetched because it is corrupted.", pfex);
		}
	}


	@Override
	public TablePage[] readPagesFromResource(byte[][] buffers, int firstPageNumber) throws IOException {

		if (buffers.length <= 0) {
			throw new IllegalArgumentException("At least one buffer should be provided.");
		}

		if (Constants.DEBUG_CHECK) {

			if (firstPageNumber < this.firstDataPageNumber) {
				throw new IOException("Page number " + firstPageNumber + " is not valid. " + "First data page is " + this.firstDataPageNumber + ".");
			}

			for (int i = 0; i < buffers.length; i++) {

				if (buffers[i].length != this.pageSize) {
					throw new IllegalArgumentException("Buffer is not big enough to hold a page.");
				}
			}
		}

		// seek and read the buffer
		ByteBuffer[] b = new ByteBuffer[buffers.length];
		for (int i = 0; i < buffers.length; i++) {
			b[i] = ByteBuffer.wrap(buffers[i], 0, this.pageSize);
		}
		int currFirstBuffer = 0;
		try {
			this.ioChannel.position(this.pageSize * (long) firstPageNumber);
			long totalSize = buffers.length * this.pageSize;
			long bytesRemaining = buffers.length * this.pageSize;

			do {




				bytesRemaining -= this.ioChannel.read(b, currFirstBuffer, buffers.length - currFirstBuffer);
				currFirstBuffer = (int) ((totalSize - bytesRemaining) / this.pageSize);
			} while (bytesRemaining > 0);
		} catch (IOException ioex) {
			throw new IOException("Page sequence [" + firstPageNumber + ", " + (firstPageNumber + buffers.length - 1) + "] could not be read from table file.",
				ioex);
		}


		TablePage[] pages = new TablePage[buffers.length];
		for (int i = 0; i < buffers.length; i++) {
			try {
				pages[i] = pageFactory.createTablePage(this.schema, buffers[i]);

			} catch (PageFormatException pfex) {
				throw new IOException("Page could not be fetched because it is corrupted.", pfex);
			}
		}

		return pages;
	}


	@Override
	public boolean equals(Object o) {

		return this == o;
	}


	@Override
	public int hashCode() {
		return System.identityHashCode(this);
	}




	public static TableResourceManager openTable(File tableFile) throws IOException, PageFormatException {
		if (tableFile == null) {
			throw new NullPointerException("Table file must not be null.");
		}

		try {

			if (!tableFile.exists()) {
				throw new IOException("Table file '" + tableFile.getCanonicalPath() + "' does not exist.");
			}

			RandomAccessFile raf = new RandomAccessFile(tableFile, "rwd");
			return new TableResourceManager(raf);
		} catch (SecurityException sex) {
			throw new IOException("The user running the system has insufficient privileges for file manipulation.");
		}
	}


	public static TableResourceManager createTable(File tableFile, TableSchema schema) throws IOException {
		if (tableFile == null) {
			throw new NullPointerException("Table file must not be null.");
		}
		if (schema == null) {
			throw new NullPointerException("Table schema must not be null.");
		}

		try {

			if (!tableFile.exists()) {
				tableFile.createNewFile();
			}

			RandomAccessFile raf = new RandomAccessFile(tableFile, "rwd");
			return new TableResourceManager(raf, schema);
		} catch (SecurityException sex) {
			throw new IOException("The user running the system has insufficient privileges for file manipulation.");
		}
	}


	public static void deleteTable(File tableFile) throws IOException {
		try {
			if (!tableFile.exists()) {
				throw new IOException("Table file '" + tableFile.getCanonicalPath() + "' does not exist exist.");
			}

			tableFile.delete();
		} catch (SecurityException sex) {
			throw new IOException("The user running the system has insufficient privileges for file manipulation.");
		}
	}


	private static TableSchema readTableHeader(FileChannel channel) throws IOException, PageFormatException {
		ByteBuffer buffer = ByteBuffer.allocate(4096);
		buffer.order(ByteOrder.LITTLE_ENDIAN);

		readIntoBuffer(channel, buffer, 16);

		if (buffer.getInt() != TABLE_HEADER_MAGIC_NUMBER) {
			throw new PageFormatException("Table header invalid. Magic number not found.");
		}
		if (buffer.getInt() != 0) {
			throw new PageFormatException("Unknown table format version.");
		}

		TableSchema schema = null;
		int pageSize = buffer.getInt();
		try {
			schema = new TableSchema(pageSize);
		} catch (IllegalArgumentException iaex) {
			throw new PageFormatException("Table header specified an unsupported page size: " + pageSize);
		}

		int numCols = buffer.getInt();
		if (numCols <= 0 || numCols > Constants.MAX_COLUMNS_IN_TABLE) {
			throw new PageFormatException("Number of columns out of range: " + numCols);
		}

		for (int i = 0; i < numCols; i++) {

			readIntoBuffer(channel, buffer, 16);


			int typeNum = buffer.getInt();
			if (typeNum < 0 || typeNum >= BasicType.values().length) {
				throw new PageFormatException("Invalid data type given for column " + i + ": " + typeNum);
			}
			BasicType basicType = BasicType.values()[typeNum];


			int length = buffer.getInt();
			if (basicType.isArrayType() && (length <= 0 || length > Constants.MAXIMAL_CHAR_ARRAY_LENGTH)) {
				throw new PageFormatException("Length for column " + i + "is out of bounds: " + length);
			}

			DataType type = DataType.get(basicType, length);

			int attributes = buffer.getInt();
			boolean nullable = (attributes & TABLE_HEADER_COLUMN_ATTRIBUTE_NULLABLE_MASK) != 0;
			boolean unique = (attributes & TABLE_HEADER_COLUMN_ATTRIBUTE_UNIQUE_MASK) != 0;

			int nameLength = buffer.getInt();
			if (nameLength < 0 || nameLength >= Constants.MAX_COLUMN_NAME_LENGTH) {
				throw new PageFormatException("Length for column " + i + "'s name is out of bounds: " + nameLength);
			}
			readIntoBuffer(channel, buffer, 2 * nameLength);
			StringBuilder bld = new StringBuilder(nameLength);
			for (int c = 0; c < nameLength; c++) {
				bld.append(buffer.getChar());
			}

			schema.addColumn(ColumnSchema.createColumnSchema(bld.toString(), type, nullable, unique));
		}

		return schema;
	}


	private static void writeTableHeader(TableSchema schema, FileChannel channel) throws IOException {
		ByteBuffer buffer = ByteBuffer.allocate(schema.getPageSize().getNumberOfBytes());
		buffer.order(ByteOrder.LITTLE_ENDIAN);

		buffer.putInt(TABLE_HEADER_MAGIC_NUMBER);
		buffer.putInt(0);
		buffer.putInt(schema.getPageSize().getNumberOfBytes());
		buffer.putInt(schema.getNumberOfColumns());

		buffer.flip();
		writeBuffer(channel, buffer);
		buffer.clear();

		for (int i = 0; i < schema.getNumberOfColumns(); i++) {
			ColumnSchema cs = schema.getColumn(i);

			buffer.putInt(cs.getDataType().getBasicType().ordinal());
			buffer.putInt(cs.getDataType().getLength());
			int attribs = 0;
			if (cs.isNullable()) {
				attribs |= TABLE_HEADER_COLUMN_ATTRIBUTE_NULLABLE_MASK;
			}
			if (cs.isUnique()) {
				attribs |= TABLE_HEADER_COLUMN_ATTRIBUTE_UNIQUE_MASK;
			}
			buffer.putInt(attribs);

			String name = cs.getColumnName();
			if (name.length() == 0 || name.length() > Constants.MAX_COLUMN_NAME_LENGTH) {
				throw new IllegalArgumentException("Column name length has invalid length.");
			}
			buffer.putInt(name.length());

			for (int s = 0; s < name.length(); s++) {
				buffer.putChar(name.charAt(s));
			}

			buffer.flip();
			writeBuffer(channel, buffer);
			buffer.clear();
		}
	}


	private static void readIntoBuffer(FileChannel channel, ByteBuffer buffer, int num) throws IOException {
		buffer.clear();
		buffer.limit(num);

		long pos = channel.position();
		int read = 0;

		while (read < num) {
			int count = channel.read(buffer);
			if (count == -1) {
				throw new EOFException();
			}
			read += count;
		}

		buffer.flip();
		if (read > num) {
			buffer.limit(num);
			channel.position(pos + num);
		}
	}


	private static final void readIntoBuffer(FileChannel channel, ByteBuffer buffer, long position, int num) throws IOException {
		buffer.clear();
		buffer.limit(num);

		int read = 0;

		while (read < num) {
			int count = channel.read(buffer, position);
			if (count == -1) {
				throw new EOFException();
			}
			read += count;
			position += count;
		}
	}


	private static void writeBuffer(FileChannel channel, ByteBuffer buffer) throws IOException {
		long bytes = buffer.remaining();
		while (bytes > 0) {
			bytes -= channel.write(buffer);
		}
	}


	private static final void writeBuffer(FileChannel channel, ByteBuffer buffer, long position) throws IOException {
		long bytes = buffer.remaining();
		while (bytes > 0) {
			int written = channel.write(buffer, position);
			bytes -= written;
			position += written;
		}
	}

}
