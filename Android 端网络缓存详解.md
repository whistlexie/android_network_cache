[TOC]



## Android 端网络缓存详解





### 网络框架http缓存的一般实现

这里拿 Volley 为例子，开展开来，进行说明，从特殊到一般，然后进行原理分析。

Volley 在构建



这个是 Volley 网络缓存实现的基类，可以看到

```java
/**
 * An interface for a cache keyed by a String with a byte array as data.
 */
public interface Cache {
    /**
     * Retrieves an entry from the cache.
     * @param key Cache key
     * @return An {@link Entry} or null in the event of a cache miss
     */
    public Entry get(String key);

    /**
     * Adds or replaces an entry to the cache.
     * @param key Cache key
     * @param entry Data to store and metadata for cache coherency, TTL, etc.
     */
    public void put(String key, Entry entry);

    /**
     * Performs any potentially long-running actions needed to initialize the cache;
     * will be called from a worker thread.
     */
    public void initialize();

    /**
     * Invalidates an entry in the cache.
     * @param key Cache key
     * @param fullExpire True to fully expire the entry, false to soft expire
     */
    public void invalidate(String key, boolean fullExpire);

    /**
     * Removes an entry from the cache.
     * @param key Cache key
     */
    public void remove(String key);

    /**
     * Empties the cache.
     */
    public void clear();

    /**
     * Data and metadata for an entry returned by the cache.
     */
    public static class Entry {
        /** The data returned from cache. */
        public byte[] data;

        /** ETag for cache coherency. */
        public String etag;

        /** Date of this response as reported by the server. */
        public long serverDate;

        /** The last modified date for the requested object. */
        public long lastModified;

        /** TTL for this record. */
        public long ttl;

        /** Soft TTL for this record. */
        public long softTtl;

        /** Immutable response headers as received from server; must be non-null. */
        public Map<String, String> responseHeaders = Collections.emptyMap();

        /** True if the entry is expired. */
        public boolean isExpired() {
            return this.ttl < System.currentTimeMillis();
        }

        /** True if a refresh is needed from the original data source. */
        public boolean refreshNeeded() {
            return this.softTtl < System.currentTimeMillis();
        }
    }

}
```

英文比较简单，我这里就不做翻译了，这个类使用一个 String 类型作为key，value 是二进制数组，保存缓存，提供了初始化方法，清除方法，主要看 Entry 类里面，这里的每一个字段都是和缓存实现相关的，具体也会对应 Http 请求体头里面的一些字段，这里把这些字段说明一下：

```java

        public byte[] data;// 二进制数组，缓存的内容
        public String etag;// 缓存的
        public long serverDate;
        public long lastModified;
        public long ttl;
        public long softTtl;
```


实现类为  DiskBasedCache 其代码如下：

```java
/*
 * Copyright (C) 2011 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package com.android.volley.toolbox;

import android.os.SystemClock;

import com.android.volley.Cache;
import com.android.volley.VolleyLog;

import java.io.BufferedInputStream;
import java.io.BufferedOutputStream;
import java.io.EOFException;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.FilterInputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.util.Collections;
import java.util.HashMap;
import java.util.Iterator;
import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Cache implementation that caches files directly onto the hard disk in the specified
 * directory. The default disk usage size is 5MB, but is configurable.
 */
public class DiskBasedCache implements Cache {

    /** Map of the Key, CacheHeader pairs */
    private final Map<String, CacheHeader> mEntries =
            new LinkedHashMap<String, CacheHeader>(16, .75f, true);

    /** Total amount of space currently used by the cache in bytes. */
    private long mTotalSize = 0;

    /** The root directory to use for the cache. */
    private final File mRootDirectory;

    /** The maximum size of the cache in bytes. */
    private final int mMaxCacheSizeInBytes;

    /** Default maximum disk usage in bytes. */
    private static final int DEFAULT_DISK_USAGE_BYTES = 5 * 1024 * 1024;

    /** High water mark percentage for the cache */
    private static final float HYSTERESIS_FACTOR = 0.9f;

    /** Magic number for current version of cache file format. */
    private static final int CACHE_MAGIC = 0x20150306;

    /**
     * Constructs an instance of the DiskBasedCache at the specified directory.
     * @param rootDirectory The root directory of the cache.
     * @param maxCacheSizeInBytes The maximum size of the cache in bytes.
     */
    public DiskBasedCache(File rootDirectory, int maxCacheSizeInBytes) {
        mRootDirectory = rootDirectory;
        mMaxCacheSizeInBytes = maxCacheSizeInBytes;
    }

    /**
     * Constructs an instance of the DiskBasedCache at the specified directory using
     * the default maximum cache size of 5MB.
     * @param rootDirectory The root directory of the cache.
     */
    public DiskBasedCache(File rootDirectory) {
        this(rootDirectory, DEFAULT_DISK_USAGE_BYTES);
    }

    /**
     * Clears the cache. Deletes all cached files from disk.
     */
    @Override
    public synchronized void clear() {
        File[] files = mRootDirectory.listFiles();
        if (files != null) {
            for (File file : files) {
                file.delete();
            }
        }
        mEntries.clear();
        mTotalSize = 0;
        VolleyLog.d("Cache cleared.");
    }

    /**
     * Returns the cache entry with the specified key if it exists, null otherwise.
     */
    @Override
    public synchronized Entry get(String key) {
        CacheHeader entry = mEntries.get(key);
        // if the entry does not exist, return.
        if (entry == null) {
            return null;
        }

        File file = getFileForKey(key);
        CountingInputStream cis = null;
        try {
            cis = new CountingInputStream(new BufferedInputStream(new FileInputStream(file)));
            CacheHeader.readHeader(cis); // eat header
            byte[] data = streamToBytes(cis, (int) (file.length() - cis.bytesRead));
            return entry.toCacheEntry(data);
        } catch (IOException e) {
            VolleyLog.d("%s: %s", file.getAbsolutePath(), e.toString());
            remove(key);
            return null;
        }  catch (NegativeArraySizeException e) {
            VolleyLog.d("%s: %s", file.getAbsolutePath(), e.toString());
            remove(key);
            return null;
        } finally {
            if (cis != null) {
                try {
                    cis.close();
                } catch (IOException ioe) {
                    return null;
                }
            }
        }
    }

    /**
     * Initializes the DiskBasedCache by scanning for all files currently in the
     * specified root directory. Creates the root directory if necessary.
     */
    @Override
    public synchronized void initialize() {
        if (!mRootDirectory.exists()) {
            if (!mRootDirectory.mkdirs()) {
                VolleyLog.e("Unable to create cache dir %s", mRootDirectory.getAbsolutePath());
            }
            return;
        }

        File[] files = mRootDirectory.listFiles();
        if (files == null) {
            return;
        }
        for (File file : files) {
            BufferedInputStream fis = null;
            try {
                fis = new BufferedInputStream(new FileInputStream(file));
                CacheHeader entry = CacheHeader.readHeader(fis);
                entry.size = file.length();
                putEntry(entry.key, entry);
            } catch (IOException e) {
                if (file != null) {
                   file.delete();
                }
            } finally {
                try {
                    if (fis != null) {
                        fis.close();
                    }
                } catch (IOException ignored) { }
            }
        }
    }

    /**
     * Invalidates an entry in the cache.
     * @param key Cache key
     * @param fullExpire True to fully expire the entry, false to soft expire
     */
    @Override
    public synchronized void invalidate(String key, boolean fullExpire) {
        Entry entry = get(key);
        if (entry != null) {
            entry.softTtl = 0;
            if (fullExpire) {
                entry.ttl = 0;
            }
            put(key, entry);
        }

    }

    /**
     * Puts the entry with the specified key into the cache.
     */
    @Override
    public synchronized void put(String key, Entry entry) {
        pruneIfNeeded(entry.data.length);
        File file = getFileForKey(key);
        try {
            BufferedOutputStream fos = new BufferedOutputStream(new FileOutputStream(file));
            CacheHeader e = new CacheHeader(key, entry);
            boolean success = e.writeHeader(fos);
            if (!success) {
                fos.close();
                VolleyLog.d("Failed to write header for %s", file.getAbsolutePath());
                throw new IOException();
            }
            fos.write(entry.data);
            fos.close();
            putEntry(key, e);
            return;
        } catch (IOException e) {
        }
        boolean deleted = file.delete();
        if (!deleted) {
            VolleyLog.d("Could not clean up file %s", file.getAbsolutePath());
        }
    }

    /**
     * Removes the specified key from the cache if it exists.
     */
    @Override
    public synchronized void remove(String key) {
        boolean deleted = getFileForKey(key).delete();
        removeEntry(key);
        if (!deleted) {
            VolleyLog.d("Could not delete cache entry for key=%s, filename=%s",
                    key, getFilenameForKey(key));
        }
    }

    /**
     * Creates a pseudo-unique filename for the specified cache key.
     * @param key The key to generate a file name for.
     * @return A pseudo-unique filename.
     */
    private String getFilenameForKey(String key) {
        int firstHalfLength = key.length() / 2;
        String localFilename = String.valueOf(key.substring(0, firstHalfLength).hashCode());
        localFilename += String.valueOf(key.substring(firstHalfLength).hashCode());
        return localFilename;
    }

    /**
     * Returns a file object for the given cache key.
     */
    public File getFileForKey(String key) {
        return new File(mRootDirectory, getFilenameForKey(key));
    }

    /**
     * Prunes the cache to fit the amount of bytes specified.
     * @param neededSpace The amount of bytes we are trying to fit into the cache.
     */
    private void pruneIfNeeded(int neededSpace) {
        if ((mTotalSize + neededSpace) < mMaxCacheSizeInBytes) {
            return;
        }
        if (VolleyLog.DEBUG) {
            VolleyLog.v("Pruning old cache entries.");
        }

        long before = mTotalSize;
        int prunedFiles = 0;
        long startTime = SystemClock.elapsedRealtime();

        Iterator<Map.Entry<String, CacheHeader>> iterator = mEntries.entrySet().iterator();
        while (iterator.hasNext()) {
            Map.Entry<String, CacheHeader> entry = iterator.next();
            CacheHeader e = entry.getValue();
            boolean deleted = getFileForKey(e.key).delete();
            if (deleted) {
                mTotalSize -= e.size;
            } else {
               VolleyLog.d("Could not delete cache entry for key=%s, filename=%s",
                       e.key, getFilenameForKey(e.key));
            }
            iterator.remove();
            prunedFiles++;

            if ((mTotalSize + neededSpace) < mMaxCacheSizeInBytes * HYSTERESIS_FACTOR) {
                break;
            }
        }

        if (VolleyLog.DEBUG) {
            VolleyLog.v("pruned %d files, %d bytes, %d ms",
                    prunedFiles, (mTotalSize - before), SystemClock.elapsedRealtime() - startTime);
        }
    }

    /**
     * Puts the entry with the specified key into the cache.
     * @param key The key to identify the entry by.
     * @param entry The entry to cache.
     */
    private void putEntry(String key, CacheHeader entry) {
        if (!mEntries.containsKey(key)) {
            mTotalSize += entry.size;
        } else {
            CacheHeader oldEntry = mEntries.get(key);
            mTotalSize += (entry.size - oldEntry.size);
        }
        mEntries.put(key, entry);
    }

    /**
     * Removes the entry identified by 'key' from the cache.
     */
    private void removeEntry(String key) {
        CacheHeader entry = mEntries.get(key);
        if (entry != null) {
            mTotalSize -= entry.size;
            mEntries.remove(key);
        }
    }

    /**
     * Reads the contents of an InputStream into a byte[].
     * */
    private static byte[] streamToBytes(InputStream in, int length) throws IOException {
        byte[] bytes = new byte[length];
        int count;
        int pos = 0;
        while (pos < length && ((count = in.read(bytes, pos, length - pos)) != -1)) {
            pos += count;
        }
        if (pos != length) {
            throw new IOException("Expected " + length + " bytes, read " + pos + " bytes");
        }
        return bytes;
    }

    /**
     * Handles holding onto the cache headers for an entry.
     */
    // Visible for testing.
    static class CacheHeader {
        /** The size of the data identified by this CacheHeader. (This is not
         * serialized to disk. */
        public long size;

        /** The key that identifies the cache entry. */
        public String key;

        /** ETag for cache coherence. */
        public String etag;

        /** Date of this response as reported by the server. */
        public long serverDate;

        /** The last modified date for the requested object. */
        public long lastModified;

        /** TTL for this record. */
        public long ttl;

        /** Soft TTL for this record. */
        public long softTtl;

        /** Headers from the response resulting in this cache entry. */
        public Map<String, String> responseHeaders;

        private CacheHeader() { }

        /**
         * Instantiates a new CacheHeader object
         * @param key The key that identifies the cache entry
         * @param entry The cache entry.
         */
        public CacheHeader(String key, Entry entry) {
            this.key = key;
            this.size = entry.data.length;
            this.etag = entry.etag;
            this.serverDate = entry.serverDate;
            this.lastModified = entry.lastModified;
            this.ttl = entry.ttl;
            this.softTtl = entry.softTtl;
            this.responseHeaders = entry.responseHeaders;
        }

        /**
         * Reads the header off of an InputStream and returns a CacheHeader object.
         * @param is The InputStream to read from.
         * @throws IOException
         */
        public static CacheHeader readHeader(InputStream is) throws IOException {
            CacheHeader entry = new CacheHeader();
            int magic = readInt(is);
            if (magic != CACHE_MAGIC) {
                // don't bother deleting, it'll get pruned eventually
                throw new IOException();
            }
            entry.key = readString(is);
            entry.etag = readString(is);
            if (entry.etag.equals("")) {
                entry.etag = null;
            }
            entry.serverDate = readLong(is);
            entry.lastModified = readLong(is);
            entry.ttl = readLong(is);
            entry.softTtl = readLong(is);
            entry.responseHeaders = readStringStringMap(is);

            return entry;
        }

        /**
         * Creates a cache entry for the specified data.
         */
        public Entry toCacheEntry(byte[] data) {
            Entry e = new Entry();
            e.data = data;
            e.etag = etag;
            e.serverDate = serverDate;
            e.lastModified = lastModified;
            e.ttl = ttl;
            e.softTtl = softTtl;
            e.responseHeaders = responseHeaders;
            return e;
        }


        /**
         * Writes the contents of this CacheHeader to the specified OutputStream.
         */
        public boolean writeHeader(OutputStream os) {
            try {
                writeInt(os, CACHE_MAGIC);
                writeString(os, key);
                writeString(os, etag == null ? "" : etag);
                writeLong(os, serverDate);
                writeLong(os, lastModified);
                writeLong(os, ttl);
                writeLong(os, softTtl);
                writeStringStringMap(responseHeaders, os);
                os.flush();
                return true;
            } catch (IOException e) {
                VolleyLog.d("%s", e.toString());
                return false;
            }
        }

    }

    private static class CountingInputStream extends FilterInputStream {
        private int bytesRead = 0;

        private CountingInputStream(InputStream in) {
            super(in);
        }

        @Override
        public int read() throws IOException {
            int result = super.read();
            if (result != -1) {
                bytesRead++;
            }
            return result;
        }

        @Override
        public int read(byte[] buffer, int offset, int count) throws IOException {
            int result = super.read(buffer, offset, count);
            if (result != -1) {
                bytesRead += result;
            }
            return result;
        }
    }

    /*
     * Homebrewed simple serialization system used for reading and writing cache
     * headers on disk. Once upon a time, this used the standard Java
     * Object{Input,Output}Stream, but the default implementation relies heavily
     * on reflection (even for standard types) and generates a ton of garbage.
     */

    /**
     * Simple wrapper around {@link InputStream#read()} that throws EOFException
     * instead of returning -1.
     */
    private static int read(InputStream is) throws IOException {
        int b = is.read();
        if (b == -1) {
            throw new EOFException();
        }
        return b;
    }

    static void writeInt(OutputStream os, int n) throws IOException {
        os.write((n >> 0) & 0xff);
        os.write((n >> 8) & 0xff);
        os.write((n >> 16) & 0xff);
        os.write((n >> 24) & 0xff);
    }

    static int readInt(InputStream is) throws IOException {
        int n = 0;
        n |= (read(is) << 0);
        n |= (read(is) << 8);
        n |= (read(is) << 16);
        n |= (read(is) << 24);
        return n;
    }

    static void writeLong(OutputStream os, long n) throws IOException {
        os.write((byte)(n >>> 0));
        os.write((byte)(n >>> 8));
        os.write((byte)(n >>> 16));
        os.write((byte)(n >>> 24));
        os.write((byte)(n >>> 32));
        os.write((byte)(n >>> 40));
        os.write((byte)(n >>> 48));
        os.write((byte)(n >>> 56));
    }

    static long readLong(InputStream is) throws IOException {
        long n = 0;
        n |= ((read(is) & 0xFFL) << 0);
        n |= ((read(is) & 0xFFL) << 8);
        n |= ((read(is) & 0xFFL) << 16);
        n |= ((read(is) & 0xFFL) << 24);
        n |= ((read(is) & 0xFFL) << 32);
        n |= ((read(is) & 0xFFL) << 40);
        n |= ((read(is) & 0xFFL) << 48);
        n |= ((read(is) & 0xFFL) << 56);
        return n;
    }

    static void writeString(OutputStream os, String s) throws IOException {
        byte[] b = s.getBytes("UTF-8");
        writeLong(os, b.length);
        os.write(b, 0, b.length);
    }

    static String readString(InputStream is) throws IOException {
        int n = (int) readLong(is);
        byte[] b = streamToBytes(is, n);
        return new String(b, "UTF-8");
    }

    static void writeStringStringMap(Map<String, String> map, OutputStream os) throws IOException {
        if (map != null) {
            writeInt(os, map.size());
            for (Map.Entry<String, String> entry : map.entrySet()) {
                writeString(os, entry.getKey());
                writeString(os, entry.getValue());
            }
        } else {
            writeInt(os, 0);
        }
    }

    static Map<String, String> readStringStringMap(InputStream is) throws IOException {
        int size = readInt(is);
        Map<String, String> result = (size == 0)
                ? Collections.<String, String>emptyMap()
                : new HashMap<String, String>(size);
        for (int i = 0; i < size; i++) {
            String key = readString(is).intern();
            String value = readString(is).intern();
            result.put(key, value);
        }
        return result;
    }


}
```

这里的代码比较多，我们从功能实现上去解析这个类：

##### 创建缓存文件夹

这里指的是缓存文件夹的创建，并不是缓存文件的创建，缓存文件的创建，发生在有缓存产生的情况，后面会讲到，创建缓存文件夹的代码比较简单，这里有一点需要注意的是，缓存文件夹是通过下面这段 api 获取的

```java
context.getCacheDir()
```

这里获取的路径是内部缓存文件夹对象，这里说的内部是相对于 SD 卡，如果不存在 SD 卡的手机，另当别论，例如我的 nexus 5x 手机

执行下面 API ，得到的结果如下：

```java
String cachePath = this.getCacheDir().getAbsolutePath();
String  filePath = this.getFilesDir().getAbsolutePath();
String eCachePath = this.getExternalCacheDir().getAbsolutePath();
String  eFilePath = this.getExternalFilesDir(Environment.DIRECTORY_DCIM).getAbsolutePath();
```

1. /data/user/0/com.example.kotlon.viewlearn/cache
2. /data/user/0/com.example.kotlon.viewlearn/files
3. /storage/emulated/0/Android/data/com.example.kotlon.viewlearn/cache
4. /storage/emulated/0/Android/data/com.example.kotlon.viewlearn/files/DCIM

这里使用 内部 Cache 文件夹的目的在于，android 建议这里存放的是零时缓存文件，并且体积比较小的（建议是 1 MB 以下）的缓存文件，因为这个文件里面的文件可能会被 android 系统在低硬盘内存的情况下会被清除，同样如果应用被卸载的话，这部分缓存文件会被自动清除。但是android 官方建议，要自己管理这部分缓存，在超过缓存上限的时候进行清理，不要依赖系统或者第三方软件帮助你进行清理。

##### 初始化

初始化会在 CacheDispatcher 类里面调用，一个应用只有一个 CacheDispatcher 对象，它是一个 Thread 类，它会在发送网络请求前去读取 Cache 文件，也就是完成 Cache 的初始化操作。初始化，会在一开始的时候被调用，后序加入新的 Request 是不会触发初始化操作的，

```java
/**
 * Initializes the DiskBasedCache by scanning for all files currently in the
 * specified root directory. Creates the root directory if necessary.
 */
@Override
public synchronized void initialize() {
    if (!mRootDirectory.exists()) {// 检查缓存文件夹是否存在，默认路劲为canchedir/volley 
        if (!mRootDirectory.mkdirs()) {//如果文件夹不存在则创建，创建失败，抛出异常
            VolleyLog.e("Unable to create cache dir %s", mRootDirectory.getAbsolutePath());
        }
        return;
    }

    File[] files = mRootDirectory.listFiles(); // 取出文件夹所有文件
    if (files == null) { // 如果不存在文件，则退出初始化（不用初始化了）
        return;
    }
    for (File file : files) {
        BufferedInputStream fis = null;
        try {
            fis = new BufferedInputStream(new FileInputStream(file)); // 读取 Cache  文件
            CacheHeader entry = CacheHeader.readHeader(fis); // 将文件转换成 CacheHeader 对象
            entry.size = file.length();
            putEntry(entry.key, entry); // 将 CacheHeader 放进 LinkedHasnMap 容器中，保存起来
        } catch (IOException e) {
            if (file != null) {
               file.delete();
            }
        } finally {
            try {
                if (fis != null) {
                    fis.close();
                }
            } catch (IOException ignored) { }
        }
    }
}
```

这里进行的工作是初始化，读取缓存文件夹的每一个 cache 文件，每一个 cache 文件对应一个 CacheHeader 对象，在将 CacheHeader 对象放进 LinkedHashMap 容器中的时候，会对 key 值进行判断，如果不存在这个 key 对应的 CacheHeader 对象，则直接加入，如果存在 key 值对应的 CacheHeader 对象，则会被替换掉旧的 CacheHeader 对象。再注意到，这个方法是线程安全的，实际上，我们在初始化的时候，会把全部的 Cache 扫描，并且读取，转换成对象。

##### 写缓存

写缓存的话，调用的是 put(String key, Entry entry) ,详细的代码如下：

```java
/**
 * Puts the entry with the specified key into the cache.
 */
@Override
public synchronized void put(String key, Entry entry) {
    pruneIfNeeded(entry.data.length);
    File file = getFileForKey(key);
    try {
        BufferedOutputStream fos = new BufferedOutputStream(new FileOutputStream(file));
        CacheHeader e = new CacheHeader(key, entry);
        boolean success = e.writeHeader(fos);
        if (!success) {
            fos.close();
            VolleyLog.d("Failed to write header for %s", file.getAbsolutePath());
            throw new IOException();
        }
        fos.write(entry.data);
        fos.close();
        putEntry(key, e);
        return;
    } catch (IOException e) {
    }
    boolean deleted = file.delete();
    if (!deleted) {
        VolleyLog.d("Could not clean up file %s", file.getAbsolutePath());
    }
}
```

这里要说明的是，如何生成文件名称或者key值的，生成 key 的方法如下：

```java
private String getFilenameForKey(String key) {
    int firstHalfLength = key.length() / 2;
    String localFilename = String.valueOf(key.substring(0, firstHalfLength).hashCode());
    localFilename += String.valueOf(key.substring(firstHalfLength).hashCode());
    return localFilename;
}
```

这里的代码比较简单，就是截取前后两部分的字符串，然后取 hash 值，这里用来做为 key 生成文件名的方法如下：

```java
public String getCacheKey() {
    return mMethod + ":" + mUrl;
}
```

这里将网络请求方式的 int 数值，和请求url拼接，然后作为key 的初始计算数值，上面分段取 hash 可能仅仅是为了保证 hash 的唯一性和长度。



##### 如何命中缓存

在 Volley 的架构上，我们发送一个网络请求之前首先去 cache 对象里面查找，如果没有的话，再去硬盘的

我们从具体的一个请求出发，我们简单的使用 Volley 有以下几个步骤：

1. 使用 Volley.newRequestQueue()api获取一个 RequestQueue 对象实例
2. 创建一个 Request 对象，这里以 StringRequest 对象为例
3. 将上面创建的 StringRequest 对象加入到 RequestQueue 对象中

前面两个步骤都是一系列的构造方法，在第三步的时候，这个网络请求才算标志着被Volley开始处理，简单的来看一下 add() 方法：

```java
public <T> Request<T> add(Request<T> request) {
    // Tag the request as belonging to this queue and add it to the set of current requests.
    request.setRequestQueue(this);
    synchronized (mCurrentRequests) { 
      // 加入一个 HashSet 容器中，这个容器存储了当前未处理的 request，注意HashSet 这个数据结构
        mCurrentRequests.add(request);
    }

    // Process requests in the order they are added.
    request.setSequence(getSequenceNumber());// 根据被加入的先后顺序，排序，添加排序标记
    request.addMarker("add-to-queue");// 添加debug 的标记，标记为"add-to-queue"

    // If the request is uncacheable, skip the cache queue and go straight to the network.
    //判断是否需要进行缓存，可以由 Request.setShouldCache()方法进行设置，默认是true（需要进行缓存）
  // 如果不需要缓存，则直接把Request 加入网络请求队列，然后返回request 对象
    if (!request.shouldCache()) {
        mNetworkQueue.add(request);
        return request;
    }

    // Insert request into stage if there's already a request with the same cache key in flight.
  // 根据 CacheKey查看request是否命中等待队列，如果命中增加一个相同的 request 到相应的队列中
    synchronized (mWaitingRequests) {
        String cacheKey = request.getCacheKey();
        if (mWaitingRequests.containsKey(cacheKey)) {
            // There is already a request in flight. Queue up.
            Queue<Request<?>> stagedRequests = mWaitingRequests.get(cacheKey);
            if (stagedRequests == null) {
                stagedRequests = new LinkedList<Request<?>>();
            }
            stagedRequests.add(request);
            mWaitingRequests.put(cacheKey, stagedRequests);
            if (VolleyLog.DEBUG) {
                VolleyLog.v("Request for cacheKey=%s is in flight, putting on hold.", cacheKey);
            }
        } else {
            // Insert 'null' queue for this cacheKey, indicating there is now a request in
            // flight.
            // 直接加入 等待队列
            mWaitingRequests.put(cacheKey, null);
            mCacheQueue.add(request);// 加入缓存请求队列
        }
        return request;
    }
}
```

但是实际上，我们并没有在这里看到正在的启动一个 request，实际上，启动一个 request 是在调用第一步，Volley.newRequestQueue()的时候启动的，执行了如下代码：

```java
queue.start();
```

然后 start()方法如下：

```java
public void start() {
    stop();  // Make sure any currently running dispatchers are stopped.
    // Create the cache dispatcher and start it.
    mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
    mCacheDispatcher.start();

    // Create network dispatchers (and corresponding threads) up to the pool size.
    for (int i = 0; i < mDispatchers.length; i++) {
        NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                mCache, mDelivery);
        mDispatchers[i] = networkDispatcher;
        networkDispatcher.start();
    }
}
```

这里的 Dispatcher 都是 Thread 类，run 方法里面会不断的执行轮询（死循环）去执行 request，也就是说，只要有新的 Request 被加入到对应的两种队列中，就会执行一个 Request。

根据代码的注释我们可以总结出一张流程图（详细版，包含了 Dispatcher 里面的分析）：

//



那么从 Cache 的角度来看，一个 Request 是怎么被处理的？它又是什么时候被触发的？回到 CacheDispatcher 类中，我们去到 run() 方法里面：

```java
@Override
public void run() {
    if (DEBUG) VolleyLog.v("start new dispatcher");
    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
    mCache.initialize();//初始化
    Request<?> request;
    while (true) {
        request = null;
        try {
            request = mCacheQueue.take();
        } catch (InterruptedException e) {
            if (mQuit) {// 如果queue被取消了，则直接返回，这里的粒度比 request 的cancel 要大
                return;
            }
            continue;
        }
        try {
            request.addMarker("cache-queue-take");
            if (request.isCanceled()) {//如果 request 被取消
                request.finish("cache-discard-canceled");
                continue;
            }
            Cache.Entry entry = mCache.get(request.getCacheKey());//是否命中 cache
            if (entry == null) {//未命中 cache ，把 request 加入 network 队列
                request.addMarker("cache-miss");
                mNetworkQueue.put(request);
                continue;
            }
            if (entry.isExpired()) {//有缓存，但是已经过期
                request.addMarker("cache-hit-expired");
                request.setCacheEntry(entry);
                mNetworkQueue.put(request);
                continue;
            }
            request.addMarker("cache-hit");
            Response<?> response = request.parseNetworkResponse(
                    new NetworkResponse(entry.data, entry.responseHeaders));
            request.addMarker("cache-hit-parsed");
            if (!entry.refreshNeeded()) {
                mDelivery.postResponse(request, response);
            } else {
                // Soft-expired cache hit. We can deliver the cached response,
                // but we need to also send the request to the network for
                // refreshing.
                request.addMarker("cache-hit-refresh-needed");
                request.setCacheEntry(entry);

                // Mark the response as intermediate.
                response.intermediate = true;

                // Post the intermediate response back to the user and have
                // the delivery then forward the request along to the network.
                final Request<?> finalRequest = request;
                mDelivery.postResponse(request, response, new Runnable() {
                    @Override
                    public void run() {
                        try {
                            mNetworkQueue.put(finalRequest);
                        } catch (InterruptedException e) {
                            // Not much we can do about this.
                        }
                    }
                });
            }
        } catch (Exception e) {
            VolleyLog.e(e, "Unhandled exception %s", e.toString());
        }
    }
}
```





##### 缓存里面保存的是什么内容

一个 Cache.Entry 对象，具体的字段如下（上面已经提到过部分字段）：

```java
    public byte[] data;// 二进制数组，缓存的是一个 Response 的 RAW data
    public String etag;// 缓存的
    public long serverDate;
    public long lastModified;
    public long ttl;
    public long softTtl;
```
然后我们看下是怎么缓存这些字段的（或者说赋值）

```java
//代码段一，保存状态行和响应体
entity.setContent(inputStream);
entity.setContentLength(connection.getContentLength());
entity.setContentEncoding(connection.getContentEncoding());
entity.setContentType(connection.getContentType());
//代码段二，保存响应头
for (Entry<String, List<String>> header : connection.getHeaderFields().entrySet()) {
if (header.getKey() != null) {
 Header h = new BasicHeader(header.getKey(), header.getValue().get(0));
 response.addHeader(h);
  }
 }
```

我们知道，http报文（http请求报文和http响应报文）由起始行（start line），首部块（header），以及可选的主体部分（body），一个 http 响应报文的组成如下表：

<version> <status> <reason-phrase>

<headers>

<entity-body>

对应的中文解释如下：

* 版本号，状态（状态码），原因短语
* header 首部字段
* 可选的主体内容部分

所以上面的代码是把一个完整的 http 响应报文进行了缓存，这里看不到部分源码，所以没办法进行更细致的解释。

##### 释放缓存及施放算法

如果进行缓存的自我管理，这是作为一个框架需要关心的，我们应该主动去处理这个缓存，而不是等待被处理，这样的话，我们就能更好的使用和管理缓存。

