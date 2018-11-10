---
layout: post
title:  "Jetpack系列之Room"
date:   2018-11-06 13:51:54
categories: Android
tags: Room 
---

* content
{:toc}

本文记录[Android Jetpack][1]中`Architecture`部分的[Room][2]的使用。




# Room概述
[Room][3]是基于Sqlite上，并在其基础上提供了一个抽象层，更加方便的使用数据库。
[Room][4]的架构图：
![Room Architecture][]

[Room][5]包含3个核心类。

* [Database][6]：包含数据库的持有类，作为数据库的操作入口。
    注解@Database的类需要满足以下条件。
    * 抽象类并且继承于[RoomDatabase][7]
    * [@Database][8]注解中包含与数据库关联实体的列表
    * 包含一个抽象方法并返回带注释的类 [@Dao][9]
通过[ Room.databaseBuilder() ][10]或者[Room.inMemoryDatabaseBuilder()][11]来获取[Database][12]实例。
* [Entity][13]：数据库中的表。
* [Dao][14]：包含操作数据库的方法。

代码示例：

User.kt
```kotlin
@Entity(tableName = "user")//tableName可省略
data class User(
        val name: String,
        val age: Int
) {
    @PrimaryKey(autoGenerate = true)
    var id: Long? = null

    @Ignore
    var list: ArrayList<String>? = null

    var createDate = Date()

    var gender: Int? = null
}
```

UserDao.kt

```kotlin
@Dao
interface UserDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    fun addUser(user: User)

    @Delete
    fun deleteUser(user: User)

    @Query("DELETE FROM user")
    fun deleteAllUser()

    @Update
    fun updateUser(user: User)

    @Query("SELECT * FROM user")
    fun queryAllUsers(): LiveData<List<User>>

    @Query("SELECT * FROM user WHERE id=:id")
    fun querySpecifiedUser(id: Long): Flowable<User>

    @Query("DELETE FROM user WHERE id=:id")
    fun deleteSpecifiedUser(id: Long)
}
```

AppDatabase.kt

```kotlin
@Database(entities = [User::class], version = 1, exportSchema = false)
@TypeConverters(ListConverter::class, DateConverter::class)
abstract class AppDatabase : RoomDatabase() {
    companion object {

        @Volatile
        private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase =
                INSTANCE ?: synchronized(this) {
                    INSTANCE
                            ?: buildDatabase(context).also { INSTANCE = it }
                }

        private fun buildDatabase(context: Context): AppDatabase {
            val dbDir = File(Environment.getExternalStorageDirectory(), "GoogleRoomDatabase")
            if (dbDir.exists()) {
                dbDir.mkdir()
            }
            val dbFile = File(dbDir, "Sample.db")
            return Room.databaseBuilder(context.applicationContext,
                    AppDatabase::class.java, dbFile.absolutePath)
                    .build()
        }
    }
    
    abstract fun userDao(): UserDao
}
```


## 增
```kotlin
@Dao
interface UserDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    fun addUser(user: User)
}
```
```kotlin
val userDao = AppDatabase.getInstance(this).userDao()
userDao.addUser(user)
```

## 删
```kotlin
@Dao
interface UserDao {
    @Delete
    fun deleteUser(user: User)
}
```
```kotlin
val userDao = AppDatabase.getInstance(this).userDao()
userDao.deleteUser(user)
```

## 改
```kotlin
@Dao
interface UserDao {
    @Update
    fun updateUser(user: User)
}
```
```kotlin
val userDao = AppDatabase.getInstance(this).userDao()
userDao.updateUser(user)
```

## 查
```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM user")
    fun queryUsers(): Flowable<List<User>>

    @Query("SELECT * FROM user WHERE name=:name")
    fun querySameUser(name: String): List<User>
}
```
```kotlin
val userDao = AppDatabase.getInstance(this).userDao()
userDao.queryUsers()
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe({ userList ->
                // userList here
            }, {
                //TODO handle error
            })
```

## 结合LiveData，ViewModel

![RoomDemo][15]

[Demo地址][16]


## Room常见问题

### 如何添加非基础数据类型的数据
以List跟Date为例：

1，声明converter
```kotlin
class ListConverter {

    @TypeConverter
    fun listToString(list: ArrayList<String>): String {
        return Gson().toJson(list)
    }

    @TypeConverter
    fun stringToList(s: String): ArrayList<String> {
        val token = object : TypeToken<ArrayList<String>>() {}.type
        return Gson().fromJson(s, token)
    }
}

class DateConverter {

    @TypeConverter
    fun toDate(timestamp: Long?): Date? {
        return if (timestamp == null) null else Date(timestamp)
    }

    @TypeConverter
    fun toTimestamp(date: Date?): Long? {
        return date?.time
    }
}
```
2，在Database声明converter注解
```kotlin
@TypeConverters(ListConverter::class, DateConverter::class)
abstract class AppDatabase : RoomDatabase() {
    xxx
}
```

### Cannot access database on the main thread since it may potentially lock the UI for a long period of time
[Room][2]的操作默认不允许在主线程进行。
例如下面这段代码就会抛出上述异常：

```kotlin
 val userDao = AppDatabase.getInstance(this).userDao()
 userDao.addUser(User("Anna",18))
```

上述操作不能在主线程进行，如果使用了RxJava则可以这么修改：

```kotlin
    val userDao = AppDatabase.getInstance(this).userDao()
    Completable.fromAction { userDao.addUser(User("Anna", 18)) }
                    .subscribeOn(Schedulers.io())
                    .observeOn(AndroidSchedulers.mainThread())
                    .subscribe {
                       // insert success
                    }
```
或者使用其他异步的方式进行。

**如果头铁的话，可以这么设置，就能在主线程进行数据库的操作。**
初始化Room的时候进行以下设置。

```kotlin
Room.databaseBuilder(context.applicationContext,
                    AppDatabase::class.java, dbFile.absolutePath)
                    .allowMainThreadQueries()
                    .build()
```

### Migration
在进行[Room][2]数据库发生改动的时候需要进行Migration的操作。
给`user`增加一个字段`gender:Int`：

1，声明Migration

```kotlin
class AppRoomMigration(startVersion: Int, endVersion: Int) : Migration(startVersion, endVersion) {
    override fun migrate(database: SupportSQLiteDatabase) {
        //版本1迁移到版本2
        if (startVersion == 1 && endVersion == 2) {
            database.execSQL("ALTER TABLE user ADD COLUMN gender Int")
        }
    }
}
```

2，添加Migration

```kotlin
//版本更改
@Database(entities = [User::class], version = 2, exportSchema = false)
@TypeConverters(ListConverter::class, DateConverter::class)
abstract class AppDatabase : RoomDatabase() {
    companion object {

        @Volatile
        private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase =
                INSTANCE ?: synchronized(this) {
                    INSTANCE
                            ?: buildDatabase(context).also { INSTANCE = it }
                }

        private fun buildDatabase(context: Context): AppDatabase {
            val dbDir = File(Environment.getExternalStorageDirectory(), "GoogleRoomDatabase")
            if (dbDir.exists()) {
                dbDir.mkdir()
            }

            val dbFile = File(dbDir, "Sample.db")
            return Room.databaseBuilder(context.applicationContext,
                    AppDatabase::class.java, dbFile.absolutePath)
                    //添加Migration
                    .addMigrations(AppRoomMigration(1, 2))
                    .build()
        }

    }

    abstract fun userDao(): UserDao
}
```
也可以跨版本进行迁移，代码同上。


  [1]: https://www.youtube.com/watch?v=LmkKFCfmnhQ&t=42s
  [2]: https://developer.android.google.cn/topic/libraries/architecture/room
  [3]: https://developer.android.google.cn/topic/libraries/architecture/room
  [4]: https://developer.android.google.cn/topic/libraries/architecture/room
  [5]: https://developer.android.google.cn/topic/libraries/architecture/room
  [6]: https://developer.android.google.cn/reference/android/arch/persistence/room/Database
  [7]: https://developer.android.google.cn/reference/android/arch/persistence/room/RoomDatabase
  [8]: https://developer.android.google.cn/reference/android/arch/persistence/room/Database
  [9]: https://developer.android.google.cn/reference/android/arch/persistence/room/Dao
  [10]: https://developer.android.google.cn/reference/android/arch/persistence/room/Room#databaseBuilder%28android.content.Context,%20java.lang.Class%3CT%3E,%20java.lang.String%29
  [11]: https://developer.android.google.cn/reference/android/arch/persistence/room/Room#inMemoryDatabaseBuilder%28android.content.Context,%20java.lang.Class%3CT%3E%29
  [12]: https://developer.android.google.cn/reference/android/arch/persistence/room/Database
  [13]: https://developer.android.google.cn/training/data-storage/room/defining-data
  [14]: https://developer.android.google.cn/training/data-storage/room/access
  [15]: http://qfxl.oss-cn-shanghai.aliyuncs.com/images/room_demo.gif
  [16]: https://github.com/qfxl/RoomSample.aliyuncs.com/images/room_demo.gif