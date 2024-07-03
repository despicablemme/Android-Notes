# contentResolver
由provider来提供和管理内容，通过contentResolver来获取；

## 数据结构
数据存储在数据库中，一个数据有多列数据子项构成，
如：文件绝对路径，。。。等

## contentResolver
`getContentResolver`可以获取一个resolver实例；

### query查询方法
 ```
Cursor cursor = contentResolver.query(Uri uri,
                      String[] projection,
                      String selection,
                      String[] selectionArgs);
 ```
 1. uri参数代表需要query的数据类型，如图片、文件等类别，传入对应的uri常量；
 2. projection表示需要返回的内容（数据中的列column），比如Image类别，对应有data,name,等数据；
 3. selection为筛选条件，用来筛选所需的文件后缀、对应的文件名等；
 4. 多个筛选，在selection中的'?'会被args替换并筛选；

### Cursor
query方法会返回一个cursor实例，它可以视作指向一条数据的指针，指向query时得到的一系列数据的其中一条，常用方法如下：
```
// 调整指向：
cursor.moveToNext();  // 指向下一条数据
cursor.moveToFirst();  // 指向第一条数据
cursor.moveToLast();  //
cursor。moveToPosition(int position);  // 指向制订位置

String[] names = getColumnNames();  // 得到数据的所有类名，即子项条目名称
int index = cursor.getColumnIndex(...);  // 得到这条数据对应那列的索引（第几列）
String data = cursor.getString(index);  // 得到该数据这列的信息以String返回
```
