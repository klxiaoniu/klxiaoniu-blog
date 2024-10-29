---
date: 2024-01-08 20:42:57
tag: Android
---

# MediaStore使用实战：实现“0权限”操作下载目录等

## 前言

- 在以前，操作外部存储通常的做法是申请存储权限，然后通过文件路径取得File(path)，进行相关操作。随着谷歌对相关权限的收紧，[Android 10引入分区存储](https://developer.android.google.cn/about/versions/10/privacy/changes?hl=zh-cn#scoped-storage)，[Android 11强制执行](https://developer.android.google.cn/about/versions/11/privacy/storage?hl=zh-cn)，加上国内要求targetAPI必须提升到30+，存储操作需要做新的适配。

- 由于谷歌提供的文档较为简略，所以**以操作Download目录为例**撰写这篇文章。[谷歌文档](https://developer.android.com/training/data-storage/shared/media?hl=zh-cn)中提及的不止这一种，包括如下：
  > **图片**（包括照片和屏幕截图），存储在 DCIM/ 和 Pictures/ 目录中。系统将这些文件添加到 **MediaStore.Images** 表格中。
  >
  > **视频**，存储在 DCIM/、Movies/ 和 Pictures/ 目录中。系统将这些文件添加到 **MediaStore.Video** 表格中。
  >
  > **音频**文件，存储在 Alarms/、Audiobooks/、Music/、Notifications/、Podcasts/ 和 Ringtones/ 目录中。此外，系统还可以识别 Music/ 或 Movies/ 目录中的音频播放列表，以及 Recordings/ 目录中的录音。系统将这些文件添加到 **MediaStore.Audio** 表格中。Recordings/ 目录在 Android 11（API 级别 30）及更低版本中不可用。
  >
  > **下载的文件**，存储在 Download/ 目录中。在搭载 Android 10（API 级别 29）及更高版本的设备上，这些文件存储在 **MediaStore.Downloads** 表格中。此表格在 Android 9（API 级别 28）及更低版本中不可用。

## 先说结论

- 可以通过设置requestLegacyExternalStorage标记并将targetSdkVersion保持在29，申请MANAGE_EXTERNAL_STORAGE权限等办法，延续以前的代码，但这是非常不文雅的做法，强烈不推荐。
- MediaStore是谷歌推荐解决方案之一，还有保存到专属文件目录、使用SAF等方案，具体区别看[这篇文档](https://developer.android.com/training/data-storage?hl=zh-cn)。
- MediaStore API只在Android 10及以上提供，低版本系统还需要保留兼容代码。本文主要讲Android 10+的做法，低版本代码略去。
- **使用MediaStore的本质是通过指定参数用ContentResolver查询数据库，获得所需文件的Uri，然后进行操作。**

## 开始使用

### 权限

- 可能用到的几条权限：

```xml
  <!-- 需要访问其他应用创建的图片时申请 -->
  <uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />

  <!-- 需要访问其他应用创建的视频时申请 -->
  <uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />

  <!-- 需要访问其他应用创建的音频时申请 -->
  <uses-permission android:name="android.permission.READ_MEDIA_AUDIO" />

  <!-- 旧版安卓兼容 -->
  <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>

  <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"
                   android:maxSdkVersion="29" />
```

- 以我的项目为例，只需要访问**本应用**存放到下载目录的文件，因此对于Android 10+，不需要申请任何权限，对于旧版本，保留读写权限。最终配置如下：

```xml
  <uses-permission
    android:name="android.permission.WRITE_EXTERNAL_STORAGE"
    android:maxSdkVersion="28" />
  <uses-permission
    android:name="android.permission.READ_EXTERNAL_STORAGE"
    android:maxSdkVersion="28" />
```

### 通过文件路径取得文件Uri

- 这里的文件路径均以Environment.DIRECTORY_DOWNLOADS开头，也就是“Download”，且包含文件名

```kotlin
      /**
       * 通过MediaStore获取文件uri
       * @return 获取失败返回null
       */
      @RequiresApi(Build.VERSION_CODES.Q)
      fun getFileUri(context: Context, path: String): Uri? {
      	// projection代表数据库中需要检索出来的列，也可以不写，query的第二个参数传null，写了性能更好
          val projection = arrayOf(
              MediaStore.Downloads.DISPLAY_NAME,
              MediaStore.Downloads._ID,
              MediaStore.Downloads.RELATIVE_PATH
          )
          // 从path解析出路径和文件名
          val directoryPath = path.substringBeforeLast("/")
          val fileName = path.substringAfterLast("/")
  
  		// SQL语句，路径匹配和文件名匹配
          val selection =
              "${MediaStore.Downloads.RELATIVE_PATH} LIKE ? AND ${MediaStore.Downloads.DISPLAY_NAME} = ?"
          // SQL语句参数
          val selectionArgs = arrayOf("%$directoryPath%", fileName)
  
          val contentResolver: ContentResolver = context.contentResolver
          val uri = MediaStore.Downloads.EXTERNAL_CONTENT_URI
  		// 使用ContentResolver查找，获得数据库指针
          val cursor = contentResolver.query(uri, projection, selection, selectionArgs, null)
  
          var fileUri: Uri? = null
          if (cursor?.moveToFirst() == true) {
              val columnIndex = cursor.getColumnIndex(MediaStore.Downloads._ID)
              val fileId = cursor.getLong(columnIndex)
              fileUri = Uri.withAppendedPath(uri, fileId.toString())
              cursor.close()
          }
          return fileUri
      }
  
```

### 向Download写文件

- 这里的inputStream可以通过File#inputStream()获得，也可以是其他形式的inputStream，如okhttp下载文件。提供灵活性。
- path支持直接嵌套目录。

```kotlin
    /**
     * 写inputStream到公共目录Download
     * @param path 文件路径，必须以Download/开头，且不包含文件名
     */
    @RequiresApi(Build.VERSION_CODES.Q)
    fun writeToDownload(
        context: Context,
        path: String,
        fileName: String,
        inputStream: InputStream
    ) {
        val contentValues = ContentValues()
        // 设置文件路径
        contentValues.put(MediaStore.Downloads.RELATIVE_PATH, path)
        // 设置文件名称
        contentValues.put(MediaStore.Downloads.DISPLAY_NAME, fileName)
        // ContentUri 表示操作哪个数据库, contentValues 表示要插入的数据内容
        val uri = context.contentResolver.insert(
            MediaStore.Downloads.EXTERNAL_CONTENT_URI,
            contentValues
        )!!
        // 向 path/filename 文件中插入数据
        val os: OutputStream = context.contentResolver.openOutputStream(uri)!!
        val bos = BufferedOutputStream(os)
        inputStream.use { istream ->
            bos.use { bos ->
                val buff = ByteArray(1024)
                var count: Int
                while (istream.read(buff).apply { count = this } != -1) {
                    bos.write(buff, 0, count)
                }
            }
        }
    }

```

### 通过path检查文件存在性

```kotlin
    fun checkFileExistence(context: Context, path: String): Boolean {
        return getFileUri(context, path) != null
    }
```

### 通过Uri获取文件名

```kotlin
    fun getFileName(context: Context, uri: Uri): String {
        var fileName = ""
        val contentResolver = context.contentResolver
        // 此处只需要取出文件名这一项
        val projection = arrayOf(OpenableColumns.DISPLAY_NAME)
        contentResolver.query(uri, projection, null, null, null)?.use { cursor ->
            val nameIndex = cursor.getColumnIndex(OpenableColumns.DISPLAY_NAME)
            cursor.moveToFirst()
            fileName = cursor.getString(nameIndex)
            cursor.close()
        }

        return fileName
    }
```

### 删除指定path文件

```kotlin
    fun deleteFile(context: Context, path: String) {
        val uri = getFileUri(context, path)
        if (uri == null) {
            return
        }
        context.contentResolver.delete(uri, null, null)
    }
```

### 跳转其他程序打开文件

```kotlin
    fun openFile(context: Context, path: String) {
        val intent = Intent()
        intent.action = Intent.ACTION_VIEW
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
        intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
        val uri = getFileUri(context, path)
        if (uri == null) {
            toastError("打开文件失败")
            return
        }
        intent.setDataAndType(uri, context.contentResolver.getType(uri) ?: "*/*")
        try {
            context.startActivity(intent)
        } catch (exception: ActivityNotFoundException) {
            // 对于设定的MIME没有对应程序可打开的情况
            intent.setDataAndType(uri, "*/*")
            context.startActivity(intent)
        }
    }
```

### 附：判断是否可以使用MediaStore的一个小函数

```kotlin
    private fun useMediaStore(path: String): Boolean {
        return path.startsWith(Environment.DIRECTORY_DOWNLOADS) && Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q
    }
```

## 后记

- 这些操作都是针对文件的，MediaStore似乎没有提供针对目录的操作。
- 本文根据个人理解撰写，如有不同意见，欢迎讨论。
