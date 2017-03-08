#DiskLruCache的简单使用

防止多图OOM的核心解决思路就是使用LruCache技术。但LruCache只是管理了内存中图片的存储与释放，如果图片从内存中被移除的话，那么又需要从网络上重新加载一次图片，这显然非常耗时。对此，Google又提供了一套硬盘缓存的解决方案：DiskLruCache(非Google官方编写，但获得官方认证)

一般应用都会将缓存的位置选择为`/sdcard/Android/data/<application package>/cache`这个路径，我们进入这个路径会看到很多文件夹（或者没有做分类的只有文件）。我们进入一个文件夹中，最底部会看到一个journal文件。它是DiskLruCache的一个日志文件，程序对每张图片的操作记录都存放在这个文件中，基本上看到journal这个文件就标志着该程序使用DiskLruCache技术了。

##打开缓存

DiskLruCache是不能new出实例的，如果我们要创建一个DiskLruCache的实例，则需要调用它的open()方法：
```
public static DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize)
```
open()方法接收四个参数，第一个参数指定的是**数据的缓存地址**，第二个参数指定**当前应用程序的版本号**，第三个参数指定**同一个key可以对应多少个缓存文件**，基本都是传1，第四个参数指定**最多可以缓存多少字节的数据**。

其中缓存地址的一般都是这样获取：
```
public File getDiskCacheDir(Context context, String uniqueName) {
	String cachePath;
	if (Environment.MEDIA_MOUNTED.equals(Environment.getExternalStorageState())
			|| !Environment.isExternalStorageRemovable()) {
		cachePath = context.getExternalCacheDir().getPath();
	} else {
		cachePath = context.getCacheDir().getPath();
	}
	return new File(cachePath + File.separator + uniqueName);
}
```

当SD卡存在或者SD卡不可被移除的时候，就调用getExternalCacheDir()方法来获取缓存路径，否则就调用getCacheDir()方法来获取缓存路径。前者获取到的就是`/sdcard/Android/data/<application package>/cache `这个路径，而后者获取到的是`/data/data/<application package>/cache`这个路径。

接着又将获取到的路径和一个uniqueName进行拼接，作为最终的缓存路径返回。那么这个uniqueName又是什么呢？其实这就是为了对不同类型的数据进行区分而设定的一个唯一值，比如说网易新闻缓存路径下分类放置的bitmap、object等文件夹。

**每当版本号改变，缓存路径下存储的所有数据都会被清除掉，因为DiskLruCache认为当应用程序有版本更新的时候，所有的数据都应该从网上重新获取。**

后面两个参数就没什么需要解释的了，第三个参数传1，第四个参数通常传入10M的大小就够了，这个可以根据自身的情况进行调节。

一个非常标准的open()方法可以这样写：
```
DiskLruCache mDiskLruCache = null;
try {
	File cacheDir = getDiskCacheDir(context, "bitmap");
	if (!cacheDir.exists()) {
		cacheDir.mkdirs();
	}
	mDiskLruCache = DiskLruCache.open(cacheDir, getAppVersion(context), 1, 10 * 1024 * 1024);
} catch (IOException e) {
	e.printStackTrace();
}

public int getAppVersion(Context context) {
	try {
			PackageInfo info = context.getPackageManager().getPackageInfo(context.getPackageName(), 0);
			return info.versionCode;
		} catch (NameNotFoundException e) {
			e.printStackTrace();
		}
		return 1;
	}
```

##写入缓存

从网络上获取图片，并且写入本地的方法

```
private boolean downloadUrlToStream(String urlString, OutputStream outputStream) {
	HttpURLConnection urlConnection = null;
	BufferedOutputStream out = null;
	BufferedInputStream in = null;
	try {
		final URL url = new URL(urlString);
		urlConnection = (HttpURLConnection) url.openConnection();
		in = new BufferedInputStream(urlConnection.getInputStream(), 8 * 1024);
		out = new BufferedOutputStream(outputStream, 8 * 1024);
		int b;
		while ((b = in.read()) != -1) {
			out.write(b);
		}
		return true;
	} catch (final IOException e) {
		e.printStackTrace();
	} finally {
		if (urlConnection != null) {
			urlConnection.disconnect();
		}
		try {
			if (out != null) {
				out.close();
			}
			if (in != null) {
				in.close();
			}
		} catch (final IOException e) {
			e.printStackTrace();
		}
	}
	return false;
}
```
有了这个方法之后，下面我们就可以使用DiskLruCache来进行写入了，写入的操作是借助DiskLruCache.Editor这个类完成的。类似地，这个类也是不能new的，需要调用DiskLruCache的edit()方法来获取实例
```
public Editor edit(String key) throws IOException
```

edit()方法接收一个参数key，这个key将会成为缓存文件的文件名，并且必须要和图片的URL是一一对应的。那么怎样才能让key和图片的URL能够一一对应呢？直接使用URL来作为key？不太合适，因为图片URL中可能包含一些特殊字符，这些字符有可能在命名文件时是不合法的。其实最简单的做法就是将图片的URL进行MD5编码，编码后的字符串肯定是唯一的，并且只会包含0-F这样的字符，完全符合文件的命名规则。
```
public String hashKeyForDisk(String key) {
	String cacheKey;
	try {
		final MessageDigest mDigest = MessageDigest.getInstance("MD5");
		mDigest.update(key.getBytes());
		cacheKey = bytesToHexString(mDigest.digest());
	} catch (NoSuchAlgorithmException e) {
		cacheKey = String.valueOf(key.hashCode());
	}
	return cacheKey;
}

private String bytesToHexString(byte[] bytes) {
	StringBuilder sb = new StringBuilder();
	for (int i = 0; i < bytes.length; i++) {
		String hex = Integer.toHexString(0xFF & bytes[i]);
		if (hex.length() == 1) {
			sb.append('0');
		}
		sb.append(hex);
	}
	return sb.toString();
}
```

可以这样写来得到一个DiskLruCache.Editor的实例：
```
String imageUrl = "http://XXXXXX.jpg";
String key = hashKeyForDisk(imageUrl);
DiskLruCache.Editor editor = mDiskLruCache.edit(key);
```

有了DiskLruCache.Editor的实例之后，我们可以调用它的newOutputStream()方法来创建一个输出流，然后把它传入到downloadUrlToStream()中就能实现下载并写入缓存的功能了。**注意newOutputStream()方法接收一个index参数，由于前面在设置valueCount的时候指定的是1，所以这里index传0就可以了**。在写入操作执行完之后，我们还需要调用一下`commit()`方法进行提交才能使写入生效，调用`abort()`方法的话则表示放弃此次写入。

一次完整的写入操作：
```
new Thread(new Runnable() {
	@Override
	public void run() {
		try {
			String imageUrl = "http://XXXX.jpg";
			String key = hashKeyForDisk(imageUrl);
			DiskLruCache.Editor editor = mDiskLruCache.edit(key);
			if (editor != null) {
				OutputStream outputStream = editor.newOutputStream(0);
				if (downloadUrlToStream(imageUrl, outputStream)) {
					editor.commit();
				} else {
					editor.abort();
				}
			}
			mDiskLruCache.flush();
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
}).start();
```

##读取缓存

读取的方法要比写入简单一些，主要是借助DiskLruCache的get()方法实现的：
```
public synchronized Snapshot get(String key) throws IOException
```

很明显，get()方法要求传入一个key来获取到相应的缓存数据，而这个key毫无疑问就是将图片URL进行MD5编码后的值了。
```
String imageUrl = "http:XXXXX.jpg";
String key = hashKeyForDisk(imageUrl);
DiskLruCache.Snapshot snapShot = mDiskLruCache.get(key);
```

这里获取到的是一个DiskLruCache.Snapshot对象，这个对象我们该怎么利用呢？很简单，只需要调用它的getInputStream()方法就可以得到缓存文件的输入流了。同样地，getInputStream()方法也需要传一个index参数，这里传入0就好。有了文件的输入流之后，想要把缓存图片显示到界面上就轻而易举了
```
try {
	String imageUrl = "http://XXXXX.jpg";
	String key = hashKeyForDisk(imageUrl);
	DiskLruCache.Snapshot snapShot = mDiskLruCache.get(key);
	if (snapShot != null) {
		InputStream is = snapShot.getInputStream(0);
		Bitmap bitmap = BitmapFactory.decodeStream(is);
		mImage.setImageBitmap(bitmap);
	}
} catch (IOException e) {
	e.printStackTrace();
}
```

##移除缓存

移除缓存主要是借助DiskLruCache的remove()方法实现的：
```
public synchronized boolean remove(String key) throws IOException
```
```
try {
	String imageUrl = "http://XXXXX.jpg";  
	String key = hashKeyForDisk(imageUrl);  
	mDiskLruCache.remove(key);
} catch (IOException e) {
	e.printStackTrace();
}
```

**这个方法我们并不应该经常去调用它。因为你完全不需要担心缓存的数据过多从而占用SD卡太多空间的问题，DiskLruCache会根据我们在调用open()方法时设定的缓存最大值来自动删除多余的缓存。只有你确定某个key对应的缓存内容已经过期，需要从网络获取最新数据的时候才应该调用remove()方法来移除缓存。**

##其他API

###size()

这个方法会返回当前缓存路径下所有缓存数据的总字节数，以byte为单位，如果应用程序中需要在界面上显示当前缓存数据的总大小，就可以通过调用这个方法计算出来

###flush()

这个方法用于将内存中的操作记录同步到日志文件（也就是journal文件）当中。这个方法非常重要，因为DiskLruCache能够正常工作的前提就是要依赖于journal文件中的内容。前面在讲解写入缓存操作的时候我有调用过一次这个方法，但其实并不是每次写入缓存都要调用一次flush()方法的，频繁地调用并不会带来任何好处，只会额外增加同步journal文件的时间。比较标准的做法就是在Activity的onPause()方法中去调用一次flush()方法就可以了。

###close()

这个方法用于将DiskLruCache关闭掉，是和open()方法对应的一个方法。关闭掉了之后就不能再调用DiskLruCache中任何操作缓存数据的方法，通常只应该在Activity的onDestroy()方法中去调用close()方法。

###delete()

这个方法用于将所有的缓存数据全部删除，比如手动清理缓存功能，其实只需要调用一下DiskLruCache的delete()方法就可以实现了。