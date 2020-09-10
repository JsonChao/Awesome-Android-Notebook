---

		title:  Android主流三方库源码分析（四、深入理解GreenDao源码）
		date: 2018/12/22 22:44:00   
		tags: 
		- Android主流三方库源码分析
		categories: 安卓主流三方库源码分析
		thumbnail: http://greenrobot.org/wordpress/wp-content/uploads/greenDAO-orm-320.png
---

---

## 前言

#### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

前两篇我们详细地分析了Android的网络底层框架OKHttp和封装框架Retrofit的核心源码，如果对OKHttp或Retrofit内部机制不了解的可以看看[Android主流三方库源码分析（一、深入理解OKHttp源码）](https://juejin.im/post/6844903631909552135)和[Android主流三方库源码分析（二、深入理解Retrofit源码）](https://juejin.im/post/6844903972071981064)，除了热门的网络库之外，我们还分析了使用最广泛的图片加载框架Glide的加载流程，大家读完这篇源码分析实力会有不少提升，有兴趣可以看看[Android主流三方库源码分析（三、深入理解Glide源码）](https://juejin.im/post/6844904049595121672)。本篇，我们将会来对目前Android数据库框架中性能最好的GreenDao来进行较为深入地讲解。

### 一、基本使用流程

#### 1、导入GreenDao的代码生成插件和库

    // 项目下的build.gradle
    buildscript {
        ...
        dependencies {
            classpath 'com.android.tools.build:gradle:2.3.0'
            classpath 'org.greenrobot:greendao-gradle-plugin:3.2.1' 
        }
    }
    
    // app模块下的build.gradle
    apply plugin: 'com.android.application'
    apply plugin: 'org.greenrobot.greendao'
    
    ...

    dependencies {
        ...
        compile 'org.greenrobot:greendao:3.2.0' 
    }

#### 2、创建一个实体类，这里为HistoryData

    @Entity
    public class HistoryData {
    
        @Id(autoincrement = true)
        private Long id;
    
        private long date;
    
        private String data;
    }
    
#### 3、选择ReBuild Project，HistoryData会被自动添加Set/get方法，并生成整个项目的DaoMaster、DaoSession类，以及与该实体HistoryData对应的HistoryDataDao。

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/75ea30aea6034e939918d0da2b43c9d9~tplv-k3u1fbpfcp-zoom-1.image)

    @Entity
    public class HistoryData {
    
        @Id(autoincrement = true)
        private Long id;
    
        private long date;
    
        private String data;
    
        @Generated(hash = 1371145256)
        public HistoryData(Long id, long date, String data) {
            this.id = id;
            this.date = date;
            this.data = data;
        }
    
        @Generated(hash = 422767273)
        public HistoryData() {
        }
    
        public Long getId() {
            return this.id;
        }
    
        public void setId(Long id) {
            this.id = id;
        }
    
        public long getDate() {
            return this.date;
        }
    
        public void setDate(long date) {
            this.date = date;
        }
    
        public String getData() {
            return this.data;
        }
    
        public void setData(String data) {
            this.data = data;
        }
    }
    
这里点明一下这几个类的作用：

- DaoMaster：所有Dao类的主人，负责整个库的运行，内部的静态抽象子类DevOpenHelper继承并重写了Android的SqliteOpenHelper。
- DaoSession：作为一个会话层的角色，用于生成相应的Dao对象、Dao对象的注册，操作Dao的具体对象。
- xxDao（HistoryDataDao）：生成的Dao对象，用于进行具体的数据库操作。

#### 4、获取并使用相应的Dao对象进行增删改查操作

    DaoMaster.DevOpenHelper devOpenHelper = new DaoMaster.DevOpenHelper(this, Constants.DB_NAME);
    SQLiteDatabase database = devOpenHelper.getWritableDatabase();
    DaoMaster daoMaster = new DaoMaster(database);
    mDaoSession = daoMaster.newSession();
    HistoryDataDao historyDataDao = daoSession.getHistoryDataDao();
    
    // 省略创建historyData的代码
    ...
    
    // 增
    historyDataDao.insert(historyData);
    
    // 删
    historyDataDao.delete(historyData);
    
    // 改
    historyDataDao.update(historyData);
    
    // 查
    List<HistoryData> historyDataList = historyDataDao.loadAll();
    
本篇文章将会以上述使用流程来对GreenDao的源码进行逐步分析，最后会分析下GreenDao中一些优秀的特性，让读者朋友们对GreenDao的理解有更一步的加深。

### 二、GreenDao使用流程分析

#### 1、创建数据库帮助类对象DaoMaster.DevOpenHelper

    DaoMaster.DevOpenHelper devOpenHelper = new DaoMaster.DevOpenHelper(this, Constants.DB_NAME);
    
创建GreenDao内部实现的数据库帮助类对象devOpenHelper，核心源码如下：

    public class DaoMaster extends AbstractDaoMaster {
    
        ...
    
        public static abstract class OpenHelper extends DatabaseOpenHelper {
        
        ...
        
             @Override
            public void onCreate(Database db) {
                Log.i("greenDAO", "Creating tables for schema version " + SCHEMA_VERSION);
                createAllTables(db, false);
            }
        }
        
        public static class DevOpenHelper extends OpenHelper {
        
            ...
            
            @Override
            public void onUpgrade(Database db, int oldVersion, int newVersion) {
                Log.i("greenDAO", "Upgrading schema from version " + oldVersion + " to " + newVersion + " by dropping all tables");
                dropAllTables(db, true);
                onCreate(db);
            }
        }
    }

DevOpenHelper自身实现了更新的逻辑，这里是弃置了所有的表，并且调用了OpenHelper实现的onCreate方法用于创建所有的表，其中DevOpenHelper继承于OpenHelper，而OpenHelper自身又继承于DatabaseOpenHelper，那么，这个DatabaseOpenHelper这个类的作用是什么呢？

    public abstract class DatabaseOpenHelper extends SQLiteOpenHelper {
    
        ...
        
        // 关注点1
        public Database getWritableDb() {
            return wrap(getWritableDatabase());
        }
        
        public Database getReadableDb() {
            return wrap(getReadableDatabase());
        }   
        
        protected Database wrap(SQLiteDatabase sqLiteDatabase) {
            return new StandardDatabase(sqLiteDatabase);
        }
        
        ...
        
        // 关注点2
        public Database getEncryptedWritableDb(String password) {
            EncryptedHelper encryptedHelper = checkEncryptedHelper();
            return encryptedHelper.wrap(encryptedHelper.getWritableDatabase(password));
        }
        
        public Database getEncryptedReadableDb(String password) {
            EncryptedHelper encryptedHelper = checkEncryptedHelper();
            return encryptedHelper.wrap(encryptedHelper.getReadableDatabase(password));
        }
        
        ...
        
        private class EncryptedHelper extends net.sqlcipher.database.SQLiteOpenHelper {
        
            ...
        
        
            protected Database wrap(net.sqlcipher.database.SQLiteDatabase     sqLiteDatabase) {
                return new EncryptedDatabase(sqLiteDatabase);
            }
        }
        
其实，DatabaseOpenHelper也是实现了SQLiteOpenHelper的一个帮助类，它内部可以获取到两种不同的数据库类型，一种是标准型的数据库**StandardDatabase**，另一种是加密型的数据库**EncryptedDatabase**，从以上源码可知，它们内部都通过wrap这样一个包装的方法，返回了对应的数据库类型，我们大致看一下StandardDatabase和EncryptedDatabase的内部实现。

    public class StandardDatabase implements Database {
    
        // 这里的SQLiteDatabase是android.database.sqlite.SQLiteDatabase包下的
        private final SQLiteDatabase delegate;
    
        public StandardDatabase(SQLiteDatabase delegate) {
            this.delegate = delegate;
        }
    
        @Override
        public Cursor rawQuery(String sql, String[] selectionArgs) {
            return delegate.rawQuery(sql, selectionArgs);
        }
    
        @Override
        public void execSQL(String sql) throws SQLException {
            delegate.execSQL(sql);
        }

        ...
    }
    
    public class EncryptedDatabaseStatement implements DatabaseStatement     {
    
        // 这里的SQLiteStatement是net.sqlcipher.database.SQLiteStatement包下的
        private final SQLiteStatement delegate;
    
        public EncryptedDatabaseStatement(SQLiteStatement delegate) {
            this.delegate = delegate;
        }
    
        @Override
        public void execute() {
            delegate.execute();
        }
        
        ...
    }
    
StandardDatabase和EncryptedDatabase这两个类内部都使用了**代理模式**给相同的接口添加了不同的具体实现，StandardDatabase自然是使用的Android包下的SQLiteDatabase，而EncryptedDatabaseStatement为了实现加密数据库的功能，则使用了一个叫做**sqlcipher**的数据库加密三方库，**如果你项目下的数据库需要保存比较重要的数据，则可以使用getEncryptedWritableDb方法来代替getdWritableDb方法对数据库进行加密，这样，我们之后的数据库操作则会以代理模式的形式间接地使用sqlcipher提供的API去操作数据库**。

#### 2、创建DaoMaster对象

    SQLiteDatabase database = devOpenHelper.getWritableDatabase();
    DaoMaster daoMaster = new DaoMaster(database);
    
首先，DaoMaster作为所有Dao对象的主人，它内部肯定是需要一个SQLiteDatabase对象的，因此，先由DaoMaster的帮助类对象devOpenHelper的getWritableDatabase方法得到一个标准的数据库类对象database，再由此创建一个DaoMaster对象。

    public class DaoMaster extends AbstractDaoMaster {
    
        ...
    
        public DaoMaster(SQLiteDatabase db) {
            this(new StandardDatabase(db));
        }

        public DaoMaster(Database db) {
            super(db, SCHEMA_VERSION);
            registerDaoClass(HistoryDataDao.class);
        }
        
        ...
    }
    
在DaoMaster的构造方法中，它首先执行了super(db, SCHEMA_VERSION)方法，即它的父类AbstractDaoMaster的构造方法。

    public abstract class AbstractDaoMaster {
    
        ...
    
        public AbstractDaoMaster(Database db, int schemaVersion) {
            this.db = db;
            this.schemaVersion = schemaVersion;
    
            daoConfigMap = new HashMap<Class<? extends AbstractDao<?, ?>>, DaoConfig>();
        }
        
        protected void registerDaoClass(Class<? extends AbstractDao<?, ?>> daoClass) {
            DaoConfig daoConfig = new DaoConfig(db, daoClass);
            daoConfigMap.put(daoClass, daoConfig);
        }
        
        ...
    }
    
在AbstractDaoMaster对象的构造方法中，除了记录当前的数据库对象db和版本schemaVersion之外，还创建了一个类型为**HashMap<Class<? extends AbstractDao<?, ?>>, DaoConfig>()的daoConfigMap对象用于保存每一个DAO对应的数据配置对象DaoConfig，并且Daoconfig对象存储了对应的Dao对象所必需的数据**。最后，在DaoMaster的构造方法中使用了registerDaoClass(HistoryDataDao.class)方法将HistoryDataDao类对象进行了注册，实际上，就是为HistoryDataDao这个Dao对象创建了相应的DaoConfig对象并将它放入daoConfigMap对象中保存起来。

#### 3、创建DaoSession对象

    mDaoSession = daoMaster.newSession();
    
在DaoMaster对象中使用了newSession方法新建了一个DaoSession对象。

    public DaoSession newSession() {
        return new DaoSession(db, IdentityScopeType.Session, daoConfigMap);
    }

在DaoSeesion的构造方法中，又做了哪些事情呢？

    public class DaoSession extends AbstractDaoSession {

        ...
    
        public DaoSession(Database db, IdentityScopeType type, Map<Class<?     extends AbstractDao<?, ?>>, DaoConfig>
                daoConfigMap) {
            super(db);
    
            historyDataDaoConfig = daoConfigMap.get(HistoryDataDao.class).clone();
            historyDataDaoConfig.initIdentityScope(type);
    
            historyDataDao = new HistoryDataDao(historyDataDaoConfig, this);
    
            registerDao(HistoryData.class, historyDataDao);
        }
        
        ...
    }

首先，调用了父类AbstractDaoSession的构造方法。

    public class AbstractDaoSession {
    
        ...
    
        public AbstractDaoSession(Database db) {
            this.db = db;
            this.entityToDao = new HashMap<Class<?>, AbstractDao<?, ?>>();
        }
        
        protected <T> void registerDao(Class<T> entityClass, AbstractDao<T, ?> dao) {
            entityToDao.put(entityClass, dao);
        }
        
        ...
    }
    
在AbstractDaoSession构造方法里面**创建了一个实体与Dao对象的映射集合**。接下来，在DaoSession的构造方法中还做了2件事：

- 1、**创建每一个Dao对应的DaoConfig对象**，这里是historyDataDaoConfig，**并且根据IdentityScopeType的类型初始化创建一个相应的IdentityScope**，根据type的不同，它有两种类型，分别是**IdentityScopeObject**和**IdentityScopeLong**，它的作用是根据主键缓存对应的实体数据。当主键是数字类型的时候，如long/Long、int/Integer、short/Short、byte/Byte，则使用IdentityScopeLong缓存实体数据，当主键不是数字类型的时候，则使用IdentityScopeObject缓存实体数据。
- 2、**根据DaoSession对象和每一个Dao对应的DaoConfig对象，创建与之对应的historyDataDao对象**，由于这个项目只创建了一个实体类HistoryData，因此这里只有一个Dao对象historyDataDao，然后就是注册Dao对象，其实就是将实体和对应的Dao对象放入entityToDao这个映射集合中保存起来了。

#### 4、插入源码分析

    HistoryDataDao historyDataDao = daoSession.getHistoryDataDao();
    
    // 增
    historyDataDao.insert(historyData);

这里首先在会话层DaoSession中获取了我们要操作的Dao对象HistoryDataDao，然后插入了一个我们预先创建好的historyData实体对象。其中HistoryDataDao继承了AbstractDao<HistoryData, Long> 。

    public class HistoryDataDao extends AbstractDao<HistoryData, Long> {
        ...
    }

那么，这个AbstractDao是干什么的呢？

    public abstract class AbstractDao<T, K> {
    
        ...
        
        public List<T> loadAll() {
            Cursor cursor = db.rawQuery(statements.getSelectAll(), null);
            return loadAllAndCloseCursor(cursor);
        }
        
        ...
        
        public long insert(T entity) {
            return executeInsert(entity, statements.getInsertStatement(),     true);
        }
        
        ...
        
        public void delete(T entity) {
            assertSinglePk();
            K key = getKeyVerified(entity);
            deleteByKey(key);
        }
        
        ...
    
    }
    
看到这里，根据程序员优秀的直觉，大家应该能猜到，AbstractDao是所有Dao对象的基类，它实现了实体数据的操作如增删改查。我们接着分析insert是如何实现的，在AbstractDao的insert方法中又调用了executeInsert这个方法。在这个方法中，第二个参里的statements是一个**TableStatements**对象，它是在AbstractDao初始化构造器时从DaoConfig对象中取出来的，是一个**根据指定的表格创建SQL语句的一个帮助类**。使用statements.getInsertStatement()则是获取了一个插入的语句。而第三个参数则是判断是否是主键的标志。

    public class TableStatements {
    
        ...
    
        public DatabaseStatement getInsertStatement() {
            if (insertStatement == null) {
                String sql = SqlUtils.createSqlInsert("INSERT INTO ", tablename, allColumns);
                DatabaseStatement newInsertStatement = db.compileStatement(sql);
                ...
            }
            return insertStatement;
        }
    
        ...
    }

在TableStatements的getInsertStatement方法中，主要做了两件事：
- 1、**使用SqlUtils创建了插入的sql语句**。
- 2、**根据不同的数据库类型（标准数据库或加密数据库）将sql语句编译成当前数据库对应的语句**。

我们继续往下分析executeInsert的执行流程。

    private long executeInsert(T entity, DatabaseStatement stmt, boolean setKeyAndAttach) {
        long rowId;
        if (db.isDbLockedByCurrentThread()) {
            rowId = insertInsideTx(entity, stmt);
        } else {
            db.beginTransaction();
            try {
                rowId = insertInsideTx(entity, stmt);
                db.setTransactionSuccessful();
            } finally {
                db.endTransaction();
            }
        }
        if (setKeyAndAttach) {
            updateKeyAfterInsertAndAttach(entity, rowId, true);
        }
        return rowId;
    }

这里首先是判断数据库是否被当前线程锁定，如果是，则直接插入数据，否则为了避免死锁，则开启一个数据库事务，再进行插入数据的操作。最后如果设置了主键，则在插入数据之后更新主键的值并将对应的实体缓存到相应的identityScope中，这一块的代码流程如下所示：

    protected void updateKeyAfterInsertAndAttach(T entity, long rowId, boolean lock) {
        if (rowId != -1) {
            K key = updateKeyAfterInsert(entity, rowId);
            attachEntity(key, entity, lock);
        } else {
           ...
        }
    }
    
    protected final void attachEntity(K key, T entity, boolean lock) {
        attachEntity(entity);
        if (identityScope != null && key != null) {
            if (lock) {
                identityScope.put(key, entity);
            } else {
                identityScope.putNoLock(key, entity);
            }
        }
    }
    
接着，我们还是继续追踪主线流程，在executeInsert这个方法中调用了insertInsideTx进行数据的插入。

    private long insertInsideTx(T entity, DatabaseStatement stmt) {
        synchronized (stmt) {
            if (isStandardSQLite) {
                SQLiteStatement rawStmt = (SQLiteStatement) stmt.getRawStatement();
                bindValues(rawStmt, entity);
                return rawStmt.executeInsert();
            } else {
                bindValues(stmt, entity);
                return stmt.executeInsert();
            }
        }
    }
    
为了防止并发，这里使用了悲观锁保证了数据的一致性，在AbstractDao这个类中，大量使用了这种锁保证了它的线程安全性。接着，如果当前是标准数据库，则直接获取stmt这个DatabaseStatement类对应的原始语句进行实体字段属性的绑定和最后的执行插入操作。如果是加密数据库，则直接使用当前的加密数据库所属的插入语句进行实体字段属性的绑定和执行最后的插入操作。其中bindValues这个方法对应的实现类就是我们的HistoryDataDao类。

    public class HistoryDataDao extends AbstractDao<HistoryData, Long> {
    
        ...
    
        @Override
        protected final void bindValues(DatabaseStatement stmt, HistoryData     entity) {
            stmt.clearBindings();
    
            Long id = entity.getId();
            if (id != null) {
                stmt.bindLong(1, id);
            }
            stmt.bindLong(2, entity.getDate());
    
            String data = entity.getData();
            if (data != null) {
                stmt.bindString(3, data);
            }
        }
        
        @Override
        protected final void bindValues(SQLiteStatement stmt, HistoryData     entity) {
            stmt.clearBindings();
    
            Long id = entity.getId();
            if (id != null) {
                stmt.bindLong(1, id);
            }
            stmt.bindLong(2, entity.getDate());
    
            String data = entity.getData();
            if (data != null) {
                stmt.bindString(3, data);
            }
        }
    
        ...
    }
    
可以看到，这里对HistoryData的所有字段使用对应的数据库语句进行了绑定操作。这里最后再提及一下，**如果当前数据库是加密型时，则会使用最开始提及的DatabaseStatement的加密实现类EncryptedDatabaseStatement应用代理模式去使用sqlcipher这个加密型数据库的insert方法**。

#### 5、查询源码分析

经过对插入源码的分析，我相信大家对GreenDao内部的机制已经有了一些自己的理解，由于删除和更新内部的流程比较简单，且与插入源码有异曲同工之妙，这里就不再赘述了。最后我们再分析下查询的源码，查询的流程调用链较长，所以将它的核心流程源码直接给出。

    List<HistoryData> historyDataList = historyDataDao.loadAll();
    
    public List<T> loadAll() {
        Cursor cursor = db.rawQuery(statements.getSelectAll(), null);
        return loadAllAndCloseCursor(cursor);
    }
    
    protected List<T> loadAllAndCloseCursor(Cursor cursor) {
        try {
            return loadAllFromCursor(cursor);
        } finally {
            cursor.close();
        }
    }
    
    protected List<T> loadAllFromCursor(Cursor cursor) {
        int count = cursor.getCount();
        ...
        boolean useFastCursor = false;
        if (cursor instanceof CrossProcessCursor) {
            window = ((CrossProcessCursor) cursor).getWindow();
            if (window != null) {  
                if (window.getNumRows() == count) {
                    cursor = new FastCursor(window);
                    useFastCursor = true;
                } else {
                  ...
                }
            }
        }

        if (cursor.moveToFirst()) {
            ...
            try {
                if (!useFastCursor && window != null && identityScope != null) {
                    loadAllUnlockOnWindowBounds(cursor, window, list);
                } else {
                    do {
                        list.add(loadCurrent(cursor, 0, false));
                    } while (cursor.moveToNext());
                }
            } finally {
                ...
            }
        }
        return list;
    }

最终，loadAll方法将会调用到loadAllFromCursor这个方法，首先，如果**当前的游标cursor是跨进程的cursor**，并且cursor的行数没有偏差的话，则使用一个加快版的**FastCursor**对象进行游标遍历。接着，不管是执行loadAllUnlockOnWindowBounds这个方法还是直接加载当前的数据列表list.add(loadCurrent(cursor, 0, false))，最后都会调用到这行list.add(loadCurrent(cursor, 0, false))代码，很明显，loadCurrent方法就是加载数据的方法。

    final protected T loadCurrent(Cursor cursor, int offset, boolean lock) {
        if (identityScopeLong != null) {
            ...
            T entity = lock ? identityScopeLong.get2(key) : identityScopeLong.get2NoLock(key);
            if (entity != null) {
                return entity;
            } else {
                entity = readEntity(cursor, offset);
                attachEntity(entity);
                if (lock) {
                    identityScopeLong.put2(key, entity);
                } else {
                    identityScopeLong.put2NoLock(key, entity);
                }
                return entity;
            }
        } else if (identityScope != null) {
            ...
            T entity = lock ? identityScope.get(key) : identityScope.getNoLock(key);
            if (entity != null) {
                return entity;
            } else {
                entity = readEntity(cursor, offset);
                attachEntity(key, entity, lock);
                return entity;
            }
        } else {
            ...
            T entity = readEntity(cursor, offset);
            attachEntity(entity);
            return entity;
        }
    }
    
我们来理解下loadCurrent这个方法内部的执行策略。**首先，如果有实体数据缓存identityScopeLong/identityScope，则先从缓存中取，如果缓存中没有，会使用该实体对应的Dao对象，这里的是HistoryDataDao，它在内部根据游标取出的数据新建了一个新的HistoryData实体对象返回。**

    @Override
    public HistoryData readEntity(Cursor cursor, int offset) {
        HistoryData entity = new HistoryData( //
            cursor.isNull(offset + 0) ? null : cursor.getLong(offset + 0), // id
            cursor.getLong(offset + 1), // date
            cursor.isNull(offset + 2) ? null : cursor.getString(offset + 2) // data
        );
        return entity;
    }
    
**最后，如果是非identityScopeLong缓存类型，即是属于identityScope的情况下，则还会在identityScope中将上面获得的数据进行缓存。如果没有实体数据缓存的话，则直接调用readEntity组装数据返回即可。**

注意：对于GreenDao缓存的特性，可能会出现没有拿到最新数据的bug，因此，如果遇到这种情况，可以使用DaoSession的clear方法删除缓存。

### 三、GreenDao是如何与ReactiveX结合？

首先，看下与rx结合的使用流程：

    RxDao<HistoryData, Long> xxDao = daoSession.getHistoryDataDao().rx();
    xxDao.insert(historyData)
            .observerOn(AndroidSchedulers.mainThread())
            .subscribe(new Action1<HistoryData>() {
                @Override
                public void call(HistoryData entity) {
                    // insert success
                }
            });
    
在AbstractDao对象的.rx()方法中，创建了一个默认执行在io线程的rxDao对象。

    @Experimental
    public RxDao<T, K> rx() {
        if (rxDao == null) {
            rxDao = new RxDao<>(this, Schedulers.io());
        }
        return rxDao;
    }
    
接着分析rxDao的insert方法。

    @Experimental
    public Observable<T> insert(final T entity) {
        return wrap(new Callable<T>() {
            @Override
            public T call() throws Exception {
                dao.insert(entity);
                return entity;
            }
        });
    }

起实质作用的就是这个wrap方法了，在这个方法里面主要是调用了RxUtils.fromCallable(callable)这个方法。

    @Internal
    class RxBase {
    
        ...
    
        protected <R> Observable<R> wrap(Callable<R> callable) {
            return wrap(RxUtils.fromCallable(callable));
        }
    
        protected <R> Observable<R> wrap(Observable<R> observable) {
            if (scheduler != null) {
                return observable.subscribeOn(scheduler);
            } else {
                return observable;
            }
        }
    
        ...
    }

在RxUtils的fromCallable这个方法内部，其实就是**使用defer这个延迟操作符来进行被观察者事件的发送，主要目的就是为了确保Observable被订阅后才执行**。最后，如果调度器scheduler存在的话，将通过外部的wrap方法将执行环境调度到io线程。

    @Internal
    class RxUtils {

        @Internal
        static <T> Observable<T> fromCallable(final Callable<T> callable) {
            return Observable.defer(new Func0<Observable<T>>() {
    
                @Override
                public Observable<T> call() {
                    T result;
                    try {
                        result = callable.call();
                    } catch (Exception e) {
                        return Observable.error(e);
                    }
                    return Observable.just(result);
                }
            });
        }
    }

### 四、总结

在分析完GreenDao的核心源码之后，我发现，GreenDao作为最好的数据库框架之一，是有一定道理的。**首先，它通过使用自身的插件配套相应的freemarker模板生成所需的静态代码，避免了反射等消耗性能的操作。其次，它内部提供了实体数据的映射缓存机制，能够进一步加快查询速度。对于不同数据库对应的SQL语句，也使用了不同的DataBaseStatement实现类结合代理模式进行了封装，屏蔽了数据库操作等繁琐的细节。最后，它使用了sqlcipher提供了加密数据库的功能，在一定程度确保了安全性，同时，结合RxJava，我们便能更简洁地实现异步的数据库操作**。GreenDao源码分析到这里就真的完结了，下一篇，笔者将会对RxJava的核心源码进行细致地讲解，以此能让大家对RxJava有一个更为深入的理解。


# 公众号

我的公众号 `JsonChao` 开通啦，如果您想第一时间获取最新文章和最新动态，欢迎扫描关注~

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c010f325f3614a01ad5f84bef58deb91~tplv-k3u1fbpfcp-zoom-1.image)


##### 参考链接：
---
1、GreenDao V3.2.2源码

2、[GreenDao源码分析](https://mp.weixin.qq.com/s?__biz=MzIwMTAzMTMxMg==&mid=2649492577&idx=1&sn=b35b0ef0f3769efa8c5d49fc5d60dd80&chksm=8eec879eb99b0e8881dd83cac912192df742ad547ccd274b7fccd2995edd08b095e1fd556cfe&scene=38#wechat_redirect)

3、[GreenDao源码分析](http://www.voidcn.com/article/p-ksrtulcy-brp.html)


## 赞赏

如果这个库对您有很大帮助，您愿意支持这个项目的进一步开发和这个项目的持续维护。你可以扫描下面的二维码，让我喝一杯咖啡或啤酒。非常感谢您的捐赠。谢谢！

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5fb7af8080fa4abda5e8194d7846e526~tplv-k3u1fbpfcp-zoom-1.image" width=20%><img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0ee1eb96ff58405998149e6d264442d7~tplv-k3u1fbpfcp-zoom-1.image" width=20%>
</div>


----

## Contanct Me

###  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

###  ●  微信群：

> **微信群如果不能扫码加入，麻烦大家想进微信群的朋友们，加我微信拉你进群。**

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/158fa86678694149824b81aa21a13341~tplv-k3u1fbpfcp-zoom-1.image" width=35%>
</div>
        

###  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


### About me

- #### Email: [chao.qu521@gmail.com]()
- #### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- #### 掘金: [https://juejin.im/user/4318537403878167](https://juejin.im/user/4318537403878167)
    


#### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

#### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/4318537403878167)上一起分享知识。