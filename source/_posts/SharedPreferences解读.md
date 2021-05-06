---
title: SharedPreferences解读
date: 2021-04-30 14:22:41
tags: Android
---

# SharedPreferences解读

## getSharedPreferences

我们通过这个方法拿到SP，该方法在Activity中内置，最终调用的是ContextWrapper的方法：

```java
@Override
public SharedPreferences getSharedPreferences(String name, int mode) {
	return mBase.getSharedPreferences(name, mode);
}
```

ContextWrapper里面调用的mBase的对应方法，通过百度，查到mBase其实就是传说中的`ContextImpl`：

```java
@Override
public SharedPreferences getSharedPreferences(String name, int mode) {

	if (mPackageInfo.getApplicationInfo().targetSdkVersion <
			Build.VERSION_CODES.KITKAT) {
		if (name == null) {
			name = "null";
		}
	}

	File file;
	synchronized (ContextImpl.class) {
		if (mSharedPrefsPaths == null) {
			mSharedPrefsPaths = new ArrayMap<>();
		}
		file = mSharedPrefsPaths.get(name);
		if (file == null) {
			file = getSharedPreferencesPath(name);
			
			//拿到文件后加入缓存，下次再拿文件就快了
			mSharedPrefsPaths.put(name, file);
		}
	}
	//调用重载方法执行真正的逻辑
	return getSharedPreferences(file, mode);
}
```

这个方法主要是用来获取文件，并且把文件放入ArrayMap缓存起来以备以后再次获取file可以提高效率。key就是文件的path，value就是文件映射的File对象。

然后使用文件作为参数，调用了同名的重载方法。

### 生成文件

上述代码中出现了一个`file`局部变量，它是个什么角色呢，下一步我们就要来揭开它神秘的面纱。

首先看到`Synchronized`关键字，并且充当了`ContextIml`的类锁，那么带来线程安全的同时，会使所有并发访问该类的操作阻塞。那么会不会造成UI线程卡顿呢？会，但是不明显。

`mSharedPrefsPaths = new ArrayMap<>();`，那么不用想，mSharedPrefsPaths是个ArrayMap，一定是个内存缓存。

此处代码不重要，请直接看下面文字结论。

```java
/**
*	拿到SP文件地址
**/
@Override
public File getSharedPreferencesPath(String name) {
	return makeFilename(getPreferencesDir(), name + ".xml");
}

/**
*	拿到SP所在目录
**/
@UnsupportedAppUsage
private File getPreferencesDir() {
	synchronized (mSync) {
		if (mPreferencesDir == null) {
		
			//创建File对象
			mPreferencesDir = new File(getDataDir(), "shared_prefs");
		}
		//确保文件一定存在
		return ensurePrivateDirExists(mPreferencesDir);
	}
}

/**
*	拿到数据目录，里面包括SP文件
**/
@Override
public File getDataDir() {
	if (mPackageInfo != null) {
		File res = null;
		if (isCredentialProtectedStorage()) {
			res = mPackageInfo.getCredentialProtectedDataDirFile();
		} else if (isDeviceProtectedStorage()) {
			res = mPackageInfo.getDeviceProtectedDataDirFile();
		} else {
			res = mPackageInfo.getDataDirFile();
		}

		if (res != null) {
			if (!res.exists() && android.os.Process.myUid() == android.os.Process.SYSTEM_UID) {
				Log.wtf(TAG, "Data directory doesn't exist for package " + getPackageName(),
						new Throwable());
			}
			return res;
		} else {
			throw new RuntimeException(
					"No data directory found for package " + getPackageName());
		}
	} else {
		throw new RuntimeException(
				"No package details found for package " + getPackageName());
	}
}

private File makeFilename(File base, String name) {
	if (name.indexOf(File.separatorChar) < 0) {
		final File res = new File(base, name);
		// We report as filesystem access here to give us the best shot at
		// detecting apps that will pass the path down to native code.
		BlockGuard.getVmPolicy().onPathAccess(res.getPath());
		return res;
	}
	throw new IllegalArgumentException(
			"File " + name + " contains a path separator");
}
```

上述代码没必要研究，就是经过一系列拼拼凑凑，最后拿到/data/data/包名/shared_prefs/[name].xml 文件，然后调用`ensurePrivateDirExists`方法确保文件一定存在(不存在就创建)。

### getSharedPreferences 重载方法

接下来调用了一个重载方法，主要是创建`SharedPreferenceImpl`对象，通过缓存进arrayMap，并且以file作为key一一绑定。

```java
@Override
public SharedPreferences getSharedPreferences(File file, int mode) {
	SharedPreferencesImpl sp;
	synchronized (ContextImpl.class) {
		//获取缓存，该缓存把file和SharedPreferencesImpl一一绑定
		final ArrayMap<file, sharedpreferencesimpl=""sharedpreferencesimpl""> cache = getSharedPreferencesCacheLocked();
		sp = cache.get(file);
		if (sp == null) {
			checkMode(mode);
			
			//创建SharedPreferencesImpl
			sp = new SharedPreferencesImpl(file, mode);
			//缓存SP
			cache.put(file, sp);
			return sp;
		}
	}
	if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
		getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
		
		//如果系统版本小于Android3.0，就调用此方法把File一次性读取出来
		sp.startReloadIfChangedUnexpectedly();
	}
	return sp;
}

private ArrayMap<file, sharedpreferencesimpl=""sharedpreferencesimpl""> getSharedPreferencesCacheLocked() {
	if (sSharedPrefsCache == null) {
		sSharedPrefsCache = new ArrayMap<>();
	}

	final String packageName = getPackageName();
	ArrayMap<file, sharedpreferencesimpl=""sharedpreferencesimpl""> packagePrefs = sSharedPrefsCache.get(packageName);
	if (packagePrefs == null) {
		packagePrefs = new ArrayMap<>();
		sSharedPrefsCache.put(packageName, packagePrefs);
	}

	return packagePrefs;
}
```

这个重载方法，主要是创建`SharedPreferencesImpl`对象并且缓存，File作为key，SharedPreferencesImpl作为value，保证了File和SharedPreferencesImpl的一一对应。

这里还区分了系统版本，3.0以下版本，通过`sp.startReloadIfChangedUnexpectedly();`，创建一个子线程，把文件内容全都读取到SharedPreferencesImpl中。

### SharedPreferencesImpl构造方法

```java
@UnsupportedAppUsage
SharedPreferencesImpl(File file, int mode) {
    mFile = file;
    //创建备份文件对象
    mBackupFile = makeBackupFile(file);
    mMode = mode;
    mLoaded = false;
    mMap = null;
    mThrowable = null;
    
    //开始加载
    startLoadFromDisk();
}

/**
* 创建一个同名文件，以.bak作为后缀
**/
static File makeBackupFile(File prefsFile) {
    return new File(prefsFile.getPath() + ".bak");
}
```

在构造方法中，在sp所在目录中先创建了一个以“.bak”为后缀的备份文件的对象，然后在子线程中从硬盘加载文件：

```java
/**
*在子线程加载文件
**/
@UnsupportedAppUsage
private void startLoadFromDisk() {
    synchronized (mLock) {
        mLoaded = false;
    }
    new Thread("SharedPreferencesImpl-load") {
        public void run() {
            loadFromDisk();
        }
    }.start();
}


private void loadFromDisk() {
    synchronized (mLock) {
        if (mLoaded) {
            return;
        }
        
        //如果备份文件实体存在，则从备份恢复sp文件
        if (mBackupFile.exists()) {
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }
    }


    Map<string, object="object"> map = null;
    StructStat stat = null;
    Throwable thrown = null;
    try {
        stat = Os.stat(mFile.getPath());
        if (mFile.canRead()) {
            BufferedInputStream str = null;
            try {
            
                //读取文件内容
                str = new BufferedInputStream(
                        new FileInputStream(mFile), 16 * 1024);
                
                //采用poll方式解析xml
                map = (Map<string, object="object">) XmlUtils.readMapXml(str);
            } catch (Exception e) {
                Log.w(TAG, "Cannot read " + mFile.getAbsolutePath(), e);
            } finally {
                IoUtils.closeQuietly(str);
            }
        }
    } catch (ErrnoException e) {
        // An errno exception means the stat failed. Treat as empty/non-existing by
        // ignoring.
    } catch (Throwable t) {
        thrown = t;
    }

    synchronized (mLock) {
        mLoaded = true;
        mThrowable = thrown;

        try {
            if (thrown == null) {
                if (map != null) {
                    mMap = map;
                    mStatTimestamp = stat.st_mtim;
                    mStatSize = stat.st_size;
                } else {
                    mMap = new HashMap<>();
                }
            }
        } catch (Throwable t) {
            mThrowable = t;
        } finally {
        
            //唯一唤醒锁的时机
            mLock.notifyAll();
        }
    }
}
```

### 阶段总结

`getSharedPreferences(String name,int mode)`方法用来初始化SP，主要就是创建xml文件，进而使用文件和mode创建`getSharedPreferencesImpl`对象并且返回。而在创建`getSharedPreferencesImpl`时就会开启一个子线程从硬盘解析xml文件到内存中，同时创建一个以.bat为后缀的备份文件。

当系统版本小于3.0时，在创建`getSharedPreferencesImpl`之后就把xml文件内容通过poll方式直接读取到`getSharedPreferencesImpl`中。

## Editor

### 获取
按照SP的使用流程，下一步就是要获取到`Editor`对象了，我们通过`Editor`提供的`putXXX()`方法来向SP中写入数据。

`Editor`是一个接口，提供了一些列的put虚方法。接口是不能直接调用的，所以我们调用的其实是他的实现类——`EditorImpl`，它是`getSharedPreferencesImpl`的**内部匿名类**，由于内部匿名类持有外部类的引用，那么它就可以访问外层类的成员！

```java
@Override
public Editor edit() {

	//todo 以后源码会取消这个同步阻塞
	synchronized (mLock) {
		awaitLoadedLocked();
	}
	//创建EditorImpl
	return new EditorImpl();
}

@GuardedBy("mLock")
private void awaitLoadedLocked() {
	if (!mLoaded) {
		BlockGuard.getThreadPolicy().onReadFromDisk();
	}
	//循环锁
	while (!mLoaded) {
		try {
			mLock.wait();
		} catch (InterruptedException unused) {
		}
	}
	if (mThrowable != null) {
		throw new IllegalStateException(mThrowable);
	}
}
```

每次调用`sp.editor()`方法都会循环`wait`锁，也就是说，当并发获取editor时是**同步阻塞**的。（以后会	取消阻塞）,然后创建并且返回一个`EditorImpl`对象，我们就可以使用它愉快的进行put操作了。

### EditorImpl定义和Put操作

```java
 public final class EditorImpl implements Editor {
 	//一把锁
	private final Object mEditorLock = new Object();

	//缓存
	@GuardedBy("mEditorLock")
	private final Map<string, object=""object""> mModified = new HashMap<>();

	//标志位
	@GuardedBy("mEditorLock")
	private boolean mClear = false;

	//喜闻乐见的putString
	@Override
	public Editor putString(String key, @Nullable String value) {
		synchronized (mEditorLock) {
			mModified.put(key, value);
			return this;
		}
	}
	
......
```

代码很简单，put操作其实就是对HashMap缓存进行写入，同时为了保证线程安全，加了锁。

### remove

```java
@Override
public Editor remove(String key) {
	synchronized (mEditorLock) {
		//为何不设置为null&#63;
		mModified.put(key, this);
		return this;
	}
}
```

当我看到`remove()`方法定义时，我惊呆了！mModified里面放了this，而this又持有了mModified，这不就造成循环引用了吗，不会出问题吗？希望日后能找到答案吧。

我觉得一般人都会设置null吧。

### clear

```java
@Override
public Editor clear() {
	synchronized (mEditorLock) {
		mClear = true;
		return this;
	}
}
```

`clear`方法也是非常奇怪，只是设置了一下标志位，并没有做其他的事情？难道这就完事了？

### commit

```java
@Override
public boolean commit() {
	long startTime = 0;

	MemoryCommitResult mcr = commitToMemory();
	
	SharedPreferencesImpl.this.enqueueDiskWrite(mcr, null);
	try {
		mcr.writtenToDiskLatch.await();
	} catch (InterruptedException e) {
		return false;
	} finally {
		if (DEBUG) {
			Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
					+ " committed after " + (System.currentTimeMillis() - startTime)
					+ " ms");
		}
	}
	notifyListeners(mcr);
	return mcr.writeToDiskResult;
}
```

### apply()

```java
@Override
public void apply() {
	final long startTime = System.currentTimeMillis();

	final MemoryCommitResult mcr = commitToMemory();
	final Runnable awaitCommit = new Runnable() {
			@Override
			public void run() {
				try {
					mcr.writtenToDiskLatch.await();
				} catch (InterruptedException ignored) {
				}

				if (DEBUG && mcr.wasWritten) {
					Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
							+ " applied after " + (System.currentTimeMillis() - startTime)
							+ " ms");
				}
			}
		};

	QueuedWork.addFinisher(awaitCommit);

	Runnable postWriteRunnable = new Runnable() {
			@Override
			public void run() {
				awaitCommit.run();
				QueuedWork.removeFinisher(awaitCommit);
			}
		};

	SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

	notifyListeners(mcr);
}
```

### notifyListeners

```java
private void notifyListeners(final MemoryCommitResult mcr) {
	if (mcr.listeners == null || (mcr.keysModified == null && !mcr.keysCleared)) {
		return;
	}
	if (Looper.myLooper() == Looper.getMainLooper()) {
	
		//如果当前是主线程，就遍历MemoryCommitResult中listener
		if (mcr.keysCleared && Compatibility.isChangeEnabled(CALLBACK_ON_CLEAR_CHANGE)) {
			for (OnSharedPreferenceChangeListener listener : mcr.listeners) {
				if (listener != null) {
					listener.onSharedPreferenceChanged(SharedPreferencesImpl.this, null);
				}
			}
		}
		for (int i = mcr.keysModified.size() - 1; i >= 0; i--) {
			final String key = mcr.keysModified.get(i);
			for (OnSharedPreferenceChangeListener listener : mcr.listeners) {
				if (listener != null) {
					listener.onSharedPreferenceChanged(SharedPreferencesImpl.this, key);
				}
			}
		}
	} else {
		//如果当前是子线程，就切换到主线程重新调用方法
		ActivityThread.sMainThreadHandler.post(() -> notifyListeners(mcr));
	}
}
```

### enqueueDiskWrite

首先看`SharedPreferencesImpl.this.enqueueDiskWrite(mcr, null);`干了什么：

```java
/**
*使已经提交给内存的结果排队，以将其写入磁盘。它们将按入队的顺序一次写入磁盘
**/
private void enqueueDiskWrite(final MemoryCommitResult mcr,
                                  final Runnable postWriteRunnable) {
	//由于传递进来的参数2是null，这里一定是true
	final boolean isFromSyncCommit = (postWriteRunnable == null);

	final Runnable writeToDiskRunnable = new Runnable() {
			@Override
			public void run() {
				synchronized (mWritingToDiskLock) {
					writeToFile(mcr, isFromSyncCommit);
				}
				synchronized (mLock) {
					mDiskWritesInFlight--;
				}
				if (postWriteRunnable != null) {
					postWriteRunnable.run();
				}
			}
		};

	// 在当前线程启动Runnable进行写入
	if (isFromSyncCommit) {
		boolean wasEmpty = false;
		synchronized (mLock) {
			wasEmpty = mDiskWritesInFlight == 1;
		}
		if (wasEmpty) {
			writeToDiskRunnable.run();
			return;
		}
	}

	QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit);
}
```

在这里先定义了一个`Runnable`叫做writeToDiskRunnable ，一看名字就知道是用来异步IO写磁盘的。

然后由于源码中调用此方法时传递的postWriteRunnable就是null，所以下面的if判断是必走的，那么就会在当前线程直接开始写入操作，这样就造成了一个结果：**阻塞了UI线程造成卡顿，并且数据量大还容易ANR**。

#### MemoryCommitResult

MemoryCommitResult是一个打包类，它把当前的内存缓存、内存状态、监听回调、写入结果全部装箱到一起。

```java
private static class MemoryCommitResult {

	//内存状态版本号
	final long memoryStateGeneration;
	final boolean keysCleared;
	@Nullable final List<string> keysModified;
	@Nullable final Set<onsharedpreferencechangelistener> listeners;
	final Map<string, object=""object""> mapToWriteToDisk;
	final CountDownLatch writtenToDiskLatch = new CountDownLatch(1);

	@GuardedBy("mWritingToDiskLock")
	volatile boolean writeToDiskResult = false;
	boolean wasWritten = false;

	//私有构造方法
	private MemoryCommitResult(long memoryStateGeneration, boolean keysCleared,
			@Nullable List<string> keysModified,
			@Nullable Set<onsharedpreferencechangelistener> listeners,
			Map<string, object=""object""> mapToWriteToDisk) {
		this.memoryStateGeneration = memoryStateGeneration;
		this.keysCleared = keysCleared;
		this.keysModified = keysModified;
		this.listeners = listeners;
		this.mapToWriteToDisk = mapToWriteToDisk;
	}
	
	//唯一提供的方法，用来变更标记为，储存写入结果
	void setDiskWriteResult(boolean wasWritten, boolean result) {
		this.wasWritten = wasWritten;
		writeToDiskResult = result;
		writtenToDiskLatch.countDown();
	}
}
```

#### 最重要的方法writeToFile

##### 备份文件

```java
// 判断是否满足写入条件，并且备份文件
if (fileExists) {
	boolean needsWrite = false;

	// 如果硬盘状态版本号低于内存状态版本号，那么只需要写入
	if (mDiskStateGeneration < mcr.memoryStateGeneration) {
		if (isFromSyncCommit) {
			needsWrite = true;//变更标志位
		} else {
			synchronized (mLock) {
				//commit方法直接来这里
				//无需保持中间状态。只需等待最新状态保持不变即可。
				if (mCurrentMemoryStateGeneration == mcr.memoryStateGeneration) {
					needsWrite = true;//变更标志位
				}
			}
		}
	}

	//如果不需要写入，那么直接设置结果
	if (!needsWrite) {
		mcr.setDiskWriteResult(false, true);
		return;
	}

	//判定备份文件是否存在
	boolean backupFileExists = mBackupFile.exists();

	//如果备份不存在，则把文件内容备份
	if (!backupFileExists) {
		if (!mFile.renameTo(mBackupFile)) {
			mcr.setDiskWriteResult(false, false);
			return;
		}
	} else {
		//如果已存在备份，则删掉文件
		mFile.delete();
	}
}
```


##### 文件不存在


##### 真正的写入

```java
// 尝试写入文件，删除备份并尽可能自动地返回true。如果发生任何异常，请删除新文件。下次我们将从备份中还原。
try {
	FileOutputStream str = createFileOutputStream(mFile);

	//如果拿不到输出流就直接返回结果：失败
	if (str == null) {
		mcr.setDiskWriteResult(false, false);
		return;
	}
	
	//执行写入操作
	XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);

	//保存写入的时间点
	writeTime = System.currentTimeMillis();

	FileUtils.sync(str);

	fsyncTime = System.currentTimeMillis();

	//写完了，关闭输出流
	str.close();
	
	//设置文件权限
	ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);

	try {
		//获取文件系统中的文件状态
		final StructStat stat = Os.stat(mFile.getPath());
		synchronized (mLock) {
			mStatTimestamp = stat.st_mtim;
			mStatSize = stat.st_size;
		}
	} catch (ErrnoException e) {
		// Do nothing
	}

	// 写入成功后删除备份
	mBackupFile.delete();

	mDiskStateGeneration = mcr.memoryStateGeneration;

	//返回结果
	mcr.setDiskWriteResult(true, true);

	long fsyncDuration = fsyncTime - writeTime;
	mSyncTimes.add((int) fsyncDuration);
	mNumSync++;

	return;
} catch (XmlPullParserException e) {
	Log.w(TAG, "writeToFile: Got exception:", e);
} catch (IOException e) {
	Log.w(TAG, "writeToFile: Got exception:", e);
}

// 清理干净没有写入成功的文件
if (mFile.exists()) {
	if (!mFile.delete()) {
		Log.e(TAG, "Couldn't clean up partially-written file " + mFile);
	}
}
//返回结果：失败
mcr.setDiskWriteResult(false, false);
```

这段逻辑真的好长，最后回顾起来却很简单，它只干了3件事情：

- 用xml文件生成一个备份，用来做失败还原
- 用XmlUtil执行写入，写入成功后删除备份返回结果
- 写入失败则清理文件，保留备份



