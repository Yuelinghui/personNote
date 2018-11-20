# Room框架的简单使用

## Room框架简介

Room库提供了SQLite的抽象操作，允许我们使用数据库访问数据，同时可以发挥出SQLite的全部力量。其实就是提供了一种让程序员可以更简单，更方便的操作数据库的方法。

## Room框架的使用

* 新建一个项目（**好像仅支持Android Studio 3.0 以上**）

* 更新gradle文件

```
    implementation "androidx.room:room-runtime:$versions.room"
    annotationProcessor "androidx.room:room-compiler:$versions.room"
    implementation "androidx.room:room-rxjava2:$versions.room"
    implementation "androidx.lifecycle:lifecycle-runtime:$versions.lifecycle"
    implementation "androidx.lifecycle:lifecycle-extensions:$versions.lifecycle"
    annotationProcessor "androidx.lifecycle:lifecycle-compiler:$versions.lifecycle"

```

如果你的应用是kotlin支持的，那就不能使用`annotationProcessor`了，需要在gradle文件里添加

```
    apply plugin: 'kotlin-kapt'

    ....

    kapt "androidx.room:room-compiler:$versions.room"

    kapt "androidx.lifecycle:lifecycle-compiler:$versions.lifecycle"
```

* 创建实体

例子里用的是User

```
@Entity(tableName = "users")
data class User(@PrimaryKey
                @ColumnInfo(name = "userid")
                val userId: String = UUID.randomUUID().toString(),
                @ColumnInfo(name = "username")
                var userName: String)

```

这里使用注解，创建一个表，表名为“users”，表中每列有"userid"和"userName"，同时"userid"还是关键字段，初始化是UUID随机生成

* 创建DAO

创建完实体，创建DAO。其实就是对数据库的一些操作。

```
@Dao
interface UserDao {

    @Query("SELECT * FROM Users LIMIT 1")
    fun getUser():Flowable<User>

    @Query("SELECT * FROM Users WHERE userid = :id")
    fun getUserById(id:String):Flowable<User>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    fun insertUser(user: User)

    @Query("DELETE FROM Users")
    fun deleteAllUsers()
}
```

* 创建DataBase

创建一个基于User类的DataBase。

```
@Database(entities = arrayOf(User::class), version = 1)
abstract class UserDatabase : RoomDatabase() {

    abstract fun userDao(): UserDao

    companion object {

        @Volatile
        private var INSTANCE: UserDatabase? = null

        fun getInstance(context: Context): UserDatabase = INSTANCE ?: synchronized(this) {
            INSTANCE ?: buildDatabase(context).also { INSTANCE = it }
        }


        private fun buildDatabase(context: Context) = Room.databaseBuilder(context.applicationContext,
                UserDatabase::class.java, "Simple.db").build()
    }
}
```

使用注解，表示这为Room数据库，声明属于数据库的实体并设置版本号，列出实体将在数据库中创建表。同时这是一个**单例**。


* 创建ViewModel和ViewModelFactory

ViewModel可以监控Activity或者Fragment的生命周期，保存UI数据。将UI显示的数据和Activity或者Fragment分开，更好的遵循了单一责任制：**Activity或者Fragment负责将数据绘制到屏幕上，同时ViewModel可以负责保存和处理所有UI需要的数据**

```
class UserViewModel(private val dataSource: UserDao) : ViewModel() {


    fun getUserName(id:String): Flowable<String> = dataSource.getUserById(id).map {
        it.userName
    }

    fun updateUserName(userId:String,userName: String): Completable {
        return Completable.fromAction {
           val user = User(userId,userName)
            dataSource.insertUser(user)
        }
    }
}
```

```
class UserModelFactory(private val dataSource:UserDao) : ViewModelProvider.Factory {

    override fun <T : ViewModel?> create(modelClass: Class<T>): T {
        if (modelClass.isAssignableFrom(UserViewModel::class.java)) {
            return UserViewModel(dataSource) as T
        }

        throw IllegalArgumentException("Unknown ViewModel class")

    }

}
```

* Activity或者Fragment使用数据

```
mViewModelFactory = Injection.provideUserModelFactory(this)
mViewModel = ViewModelProviders.of(this,mViewModelFactory).get(UserViewModel::class.java)
```

```
object Injection {

    fun provideUserDataSource(context: Context): UserDao {
        val database = UserDatabase.getInstance(context)
        return database.userDao()
    }

    fun provideUserModelFactory(context: Context): UserModelFactory {
        val dataSource = provideUserDataSource(context)
        return UserModelFactory(dataSource)
    }
}
```

## 总结

Room框架是Google推出的jetpack中的库，可以使我们更方便快捷的使用SQLite



