####一、ContentProvider简述

在Android系统中，Content Provider作为应用程序四大组件之一，它起到在应用程序之间共享数据的作用，同时它还是标准的数据访问接口。

Android系统为每个应用分配独立用户ID和组ID，结合Linux系统的文件访问权限，保护应用程序的数据。即应用程序创建数据文件时，被赋予权限的用户/应用才可以访问应用数据。

Google等网络服务商提出的开放平台，核心是开放用户数据给第三方应用使用。而用户数据是一个平台的核心竞争力，所以需要有保护的开放。Android系统的ContentProvider即为此而生。

一个大型应用程序软件框架中，垂直上一般都分为数据层、数据访问接口层以及业务应用层。数据层以文件和数据库的形式保证，甚至可以保存到网络上；数据访问层向上提供接口给业务应用，向下管理数据；业务层通过数据访问层访问数据。

Android中数据访问层由ContentProvider实现，业务应用和ContentProvider在各app进程中，数据由ContentProvider统一管理。

Android系统中，内置了轻型数据库SQLite，占用资源低，而且开源。Android系统中通讯录、信息和邮件等都是用SQLite数据库保存的。

####二、ContentProvider接口

一般提供访问功能的数据库Provider，都会提供一个接口文件。如Android系统数据库Provider应用提供的公开接口都在android.provider包内，可以在SDK的源代码目录中找到Java源码。在Android源代码frameworks/base/core/java/android/provider目录下可以找到我们定制后的文件。

文件中主要定义一些常量，如供第三方应用访问数据用的URI、MIME类型以及字段定义等。URI用来标识一种特定类型的数据，同时还要定义与之对应的资源MIME类型。例如用URI对应数据库中不同的数据表/数据集。

>URI，全称Universal Resource Identifier，即通用资源标识符，类似web中使用的http形式的URL。Android中URI通常有四部分组成：

`[content://][com.android.contacts][/contacts][/1]`
- 第一部分称为scheme，固定为content://，表示后面路径所表示的资源是ContentProvider提供的。
- 第二部分称为authority，唯一标识一个特定的ContentProvider，一般使用ContentProvider所在的包名，保证唯一性。
- 第三部分称为资源路径，表示请求的资源类型，是可选的。如果所实现的Provider只提供一种类型的资源访问，就可以省略，如果提供多种类型的资源访问，就不可省略了。
- 第四部分称为资源ID，表示请求一个特定的资源，一般是数字，唯一的标识某种资源下一个特定的实例。

>MIME类型由两部分组成，前面是数据的大类别，后面定义具体种类。例如“vnd.android.cursor.item/contact”、“vnd.android.cursor.dir/contact”，前者表示单条contact类型的数据，后者表示多条contact类型的数据。

对于使用者来说，这些接口定义是编码的主要参考，同时数据库开发者一般会提供接口使用的说明文档，甚至示例代码。对于Android原生Content Provider来说，最好的文档就是SDK，示例代码可以参考开源的原生应用代码。

####三、ContentProvider创建

a) 实现自己的Content Provider时，必须继承于ContentProvider类，并实现以下6个函数：
- onCreate()，用来执行一些初始化工作；
- query(Uri, String[], String, String[], String)，用来返回数据给调用者；
- insert(Uri, ContentValues)，用来插入新的数据；
- update(Uri, ContentValues, String, String[])，用来更新已有的数据；
- delete(Uri, String, String[])，用来删除数据；
- getType(Uri)，用来返回数据的MIME类型。


函数的实现参考原生应用即可。Provider的实现要注意：

1. 要为Provider类实现一个SQLiteOpenHelper的子类，辅助操作数据库。这个Helper类主要用于数据库的创建/升级/版本管理，同时保证到第一次操作数据库时再执行升级/打开数据库这些耗时操作。
2. 设置URI匹配规则。我们根据URI来操作数据库，不同的URI对于不同的操作。定义了URI匹配规则，当获得一个URI时，就能快速判断要如何操作数据库。我们使用UriMatcher来匹配URI。
```Java
private static final UriMatcher uriMatcher;
private static final int CONTACTS = 1000;
private static final int CONTACTS_ID = 1001;
...
static {
    uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
    uriMatcher.addURI(ContactsContract.AUTHORITY, "contacts", CONTACTS);
    uriMatcher.addURI(ContactsContract.AUTHORITY, "contacts/#", CONTACTS_ID);
    ...
}
public Cursor query(Uri uri, String[], String, String[], String) {
    ...
    int match = uriMatcher.match(uri);
}
```
调用UriMatcher的match方法，如果匹配到uri，就返回addURI时传入的第三个参数；如果不能匹配传入的uri时，将返回UriMatcher.NO_MATCH。URI字串中的#表示匹配任意数字。
3. 使用SQLiteQueryBuilder.query方法中，使用这个类辅助数据库查询操作，可以隐藏数据表的字段名，提供别名给第三方应用使用，使数据库表的内部设计对用户透明，方便后续扩展和维护。
```Java
private static final ProjectionMap contactsColumns = ProjectionMap.builder()
.add(“ringtone”, Contacts.CUSTOM_RINGTONE)
.add(“name”, Contacts.DISPLAY_NAME)
.add(Contacts.STARRED)
...
.build();
```
以上代码中ProjectionMap从HashMap扩展而来，专为数据库查询而设计，add方法第一个参数是展示给第三方的别名，第二个参数是数据表字段的真名；也可以不设置别名，使用一个参数的add方法。
4. 数据更新机制。执行insert、update和delete时，会导致数据库数据发生辩护，所以要通过ContentResolver的notifyChange方法通知注册监听特定Uri的ContentObserver对象，做相应的更新。query最后返回给调用者一个Cursor，在返回前，调用Cursor类的setNotificationUri(ContentResolver, Uri)方法，给Cursor自己注册一个监听，注意第二个参数uri的变化，一旦受到变化通知，就可以进行一些更新操作。

b) 然后在AndroidManifest.xml中配置后才能正常使用。其中包括Provider类对应的authority(可以多个)，提供的读写操作需要的权限声明。
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
        package="com.android.providers.contacts"
        ... >
    ...
    <application android:process="android.process.acore"
        android:label="@string/app_label"
        ... >

        <provider android:name="ContactsProvider2"
            android:authorities="contacts;com.android.contacts"
            ...
            android:readPermission="android.permission.READ_CONTACTS"
            android:writePermission="android.permission.WRITE_CONTACTS">
            ...
        </provider>
        ...
    </application>
</manifest>
```

####四、访问Content Provider提供的数据

a) 在使用数据的应用AndroidManifest.xml中声明读/写权限，读权限只能查询(query)，insert/update/delete需要写权限；
```xml
<uses-permission android:name="android.permission.READ_CONTACTS" />
<uses-permission android:name="android.permission.WRITE_CONTACTS" />
```
b) 在代码中通过Context类的方法getContentResolver()获取ContentResolver实例，并调用对应的接口query/insert/update/delete进行数据访问。具体的使用下一部分进行说明。

####五、使用数据库

ContentResolver提供从应用访问数据的接口，从(数据库)外部访问数据库必须经过它来实现。通过Context的方法getContentResolver()可以获取到对应上下文的ContentResolver对象。然后调用ContentResolver的query/insert/update/delete方法进行相应操作。

不是数据库所有的表都有接口进行操作，同一个表也可能支持不同的uri，所以使用数据集这个词。

#####a) 读取数据Query

`public final Cursor query(Uri uri, String[] projection, String selection, String[]selectionArgs, String sortOrder)`

参数uri用于定位对应的数据库以及数据集，一定要保证uri是对应ContentProvider所支持的格式，尽量直接使用第二部分中说的接口文件中的定义；projection是需要返回列名，相当于SELECT语句中SELECT和FROM间的内容；selection和selectionArgs组成查询过滤条件，即WHERE子句的内容，selectionArgs用于按顺序替换selection中的”?”；sortOrder即SORT BY子句的内容。


以上参数除了uri都可以是null，SQL中其他JOIN、LIMIT以及GROUP BY等语法无法通过query接口实现，有些情况通过SQL的技巧也有可能实现。如果必须进行某些特别的查询，Content Provider可以通过不同的Uri来提供对应的功能，或者也可以通过Uri.Builder的appendQueryParameter方法传递参数给Provider，但是这些方式都是需要修改Content Provider来实现。

返回值cursor就是结果数据集的游标，可以通过cursor遍历查询到的数据，getCount()方法返回查询结果的条数。Cursor接口使用前最好检查其有效性(是否null)；使用完要close，否则内存泄露；使用move*方法时要注意不要越界，未使用时位置为-1，移动到0/first才指向有效数据；move*失败时返回false。

cursor中只包含projection中的数据列，要获取包含最多数据列的数据集可以给projection传入null，如果仍不满足需要，可能需要多次或复杂的查询。如果不了解cursor中包含哪些列，可以通过getColumnCount()/getColumnNames()获取总列数和列名，列索引和列名可以通过getColumnName()/getColumnIndex()进行转换。

CursorLoader，AsyncQueryHandler是Android框架提供的包装类，可以把耗时的数据库操作放入异步任务中执行，操作结束后通过回调通知调用者进行后续操作。CursorLoader是3.0开始新加入的类，可以通过LoaderManager进行管理，而LoaderManager可以很好的与Activity/Fragment协同工作。

#####b) 写数据

ContentValues用于保存供ContentResolver处理的一个数据集合，类似于Map的结构，key当然不能重复。一般key保存数据库字段名，value保存对应的字段值。这样用来保存一条数据记录，用作update/delete方法的参数。另外用来在函数之间传递类似Map结构的数据也是很方便的。

- bulkInsert/ApplyBatch用于批量操作，前者用于对同一个Uri批量插入一组记录，后者用来进行事务操作，而且其中有很大的灵活度。
- ContentProviderOperation/ContentProviderResult用于数据库事务操作applyBatch()的参数和返回结果。数据库大批量写操作，甚至混合操作应该使用applyBatch()来实现。具体的使用可以参考相应的SDK文档。

#####c) 其他

- ContentObserver用于监听数据内容的变化，当收到数据内容变化的通知，可以重新获取数据，并进行界面刷新等操作。
- ContentResolver的方法registerContentObserver(Uri uri, boolean notifyForDescedents, ContentObserver observer)用来给特定uri注册监听器observer，对数据变化的处理放在observer的onChange()方法中。
- Cursor也有一个类似的方法registerContentObserver(ContentObserver observer)，监听Cursor对应的数据库变化。
>同时各自对应一个取消注册的方法，注册和取消注册要成对出现。

另外有些操作无效，可能是因为数据库不支持全部的query/insert/update/delete操作，这也是对数据库的进行保护的一种方法，比如屏蔽删除操作，就可以实现一个空的delete方法。

当应用继承ContentProvider类，并重写该类用于提供数据和存储数据的方法，就可以向其他应用共享其数据。虽然使用其他方法也可以对外共享数据，但数据访问方式会因数据存储的方式而不同，如：采用文件方式对外共享数据，需要进行文件操作读写数据；采用sharedpreferences共享数据，需要使用sharedpreferences API读写数据。而使用ContentProvider共享数据的好处是统一了数据访问方式。

Uri代表了要操作的数据，Uri主要包含了两部分信息：1.需要操作的ContentProvider ，2.对ContentProvider中的什么数据进行操作，一个Uri由以下几部分组成：
1. scheme：ContentProvider（内容提供者）的scheme已经由Android所规定为：content://。
2. 主机名（或Authority）：用于唯一标识这个ContentProvider，外部调用者可以根据这个标识来找到它。
3. 路径（path）：可以用来表示我们要操作的数据，路径的构建应根据业务而定，如下：
>- 要操作contact表中id为10的记录，可以构建这样的路径:/contact/10
>- 要操作contact表中id为10的记录的name字段， contact/10/name
>- 要操作contact表中的所有记录，可以构建这样的路径:/contact

要操作的数据不一定来自数据库，也可以是文件等存储方式，如下：操作xml文件中contact节点下的name节点，可以构建这样的路径：/contact/name
如果要把一个字符串转换成Uri，可以使用Uri类中的parse()方法，如下：
```Java
Uri uri = Uri.parse("content://com.changcheng.provider.contactprovider/contact")
```

####UriMatcher、ContentUris和ContentResolver简介

因为Uri代表了要操作的数据，所以我们很经常需要解析Uri，并从Uri中获取数据。Android系统提供了两个用于操作Uri的工具类，分别为UriMatcher 和ContentUris 。

UriMatcher用于匹配Uri，它的用法如下：
1. 首先把你需要匹配Uri路径全部给注册上，如下:
```Java
//常量UriMatcher.NO_MATCH表示不匹配任何路径的返回码(-1)。
UriMatcher uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
//如果match()方法匹配content://com.changcheng.sqlite.provider.contactprovider/contact路径，返回匹配码为1
uriMatcher.addURI(“com.changcheng.sqlite.provider.contactprovider”, “contact”, 1);//添加需要匹配uri，如果匹配就会返回匹配码
//如果match()方法匹配 content://com.changcheng.sqlite.provider.contactprovider/contact/230路径，返回匹配码为2
uriMatcher.addURI(“com.changcheng.sqlite.provider.contactprovider”, “contact/#”, 2);//#号为通配符
```
2. 注册完需要匹配的Uri后，就可以使用uriMatcher.match(uri)方法对输入的Uri进行匹配，如果匹配就返回匹配码，匹配码是调用addURI()方法传入的第三个参数，假设匹配`content://com.changcheng.sqlite.provider.contactprovider/contact`路径，返回的匹配码为1。

ContentUris：用于获取Uri路径后面的ID部分，它有两个比较实用的方法：
- withAppendedId(uri, id)用于为路径加上ID部分
- parseId(uri)方法用于从路径中获取ID部分

ContentResolver：当外部应用需要对ContentProvider中的数据进行添加、删除、修改和查询操作时，可以使用ContentResolver 类来完成，要获取ContentResolver 对象，可以使用Activity提供的getContentResolver()方法。 ContentResolver使用insert、delete、update、query方法，来操作数据。

ContentObserver是一个提前通知，这时候只是通知cursor说，我的内容变化了。

DataSetObserver是一个后置通知，只有通过requery() deactivate() close()方法的调用才能获得这个通知。

因此，最为重要的还是ContentObserver，它可以告诉你数据库变化了，当然如果你要在更新完Cursor的dataset之后做一些事情，datasetObserver也是必需的。
