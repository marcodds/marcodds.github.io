---
layout: post
title:  "Android文件夹总结汇总"
date:   2018-10-16 09:59:54
categories: Android
tags: Android文件夹 
---

* content
{:toc}

本文记录Android文件夹的相关知识。





# getCacheDir()、getFilesDir()
> getCacheDir() 获取的目录为/data/data/包名/cache目录

> getFilesDir()获取的目录为/data/data/包名/files，该目录为app私有的目录，sharedpreference文件，数据库文件，都存储在这里

上述方法获取到的文件夹均需root才能查看，文件夹会随着apk卸载而自动删除

# getExternalFilesDir(type)、getExternalCacheDir()
> getExternalFilesDir(type) 获取的目录为SDcard/Android/data/包名/files/type目录，其中type可以为null，该目录一般存储一些需要长时间保存的数据

> getExternalCacheDir()获取的目录为SDcard/Android/data/包名/cache，该目录一般用于存储临时数据

以上两个文件夹均会在apk卸载的时候自动删除，所以不适用于保存下载的内容。

# Environment.getExternalStorageDirectory()
> Environment.getExternalStorageDirectory()获取到的就是SD卡

# Environment.getDataDirectory()
> Environment.getDataDirectory()获取的目录为/data，需要root才能查看。

# Environment.getDownloadCacheDirectory()
> Environment.getDownloadCacheDirectory()获取的目录为/cache，需要root才能查看。

# Environment.getRootDirectory()
> Environment.getRootDirectory()获取到的为/system，需要root才能查看。

# SD卡公有目录。

**请确保有`WRITE_EXTERNAL_STORAGE`权限，这些目录不会随着apk的卸载而自动删除。**

> Environment.getExternalStoragePublicDirectory(DIRECTORY_ALARMS)，目录为/storage/sdcard0/Alarms，一般用于存储闹钟音频目录。

> Environment.getExternalStoragePublicDirectory(DIRECTORY_DCIM)，目录为/storage/sdcard0/DCIM，系统自带相机拍照录像存储目录。

> Environment.getExternalStoragePublicDirectory(DIRECTORY_DOWNLOADS)，目录为/storage/sdcard0/Download，用户下载文件目录。

> Environment.getExternalStoragePublicDirectory(DIRECTORY_MOVIES)，目录为/storage/sdcard0/Movies，影音文件存储目录。

> Environment.getExternalStoragePublicDirectory(DIRECTORY_MUSIC)，目录为/storage/sdcard0/Music，音频文件存储目录。

> Environment.getExternalStoragePublicDirectory(DIRECTORY_NOTIFICATIONS)，目录为/storage/sdcard0/Notifications，一般用于存储通知铃声目录。

> Environment.getExternalStoragePublicDirectory(DIRECTORY_PICTURES)，目录为/storage/sdcard0/Pictures，图片存储目录。

> Environment.getExternalStoragePublicDirectory(DIRECTORY_PODCASTS)，目录为/storage/sdcard0/Podcasts，播客音频存储目录。

> Environment.getExternalStoragePublicDirectory(DIRECTORY_RINGTONES)，目录为/storage/sdcard0/Ringtones，电话铃声存储文件夹。

> Environment.getExternalStoragePublicDirectory(DIRECTORY_DOCUMENTS)[api19]，目录为/storage/sdcard0/Documents，用户文档文件夹。






