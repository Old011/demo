# NotePad
# 简介
实现了NotePad的基础功能时间戳和搜索，扩展了UI美化 
# 基本要求
1.NoteList中显示条目增加时间戳显示   
2.添加笔记查询功能（根据标题查询）
# 扩展要求 
1.UI美化 
  
# 代码详解  
## 时间戳功能
### 1.添加时间戳的位置在主页面的每个列表项中添加，即在notelist_item.xml布局文件中添加一个 
```
<TextView
        android:id="@+id/text2"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:textAppearance="?android:attr/textAppearanceLarge"
        android:textSize="12dp"
        android:gravity="center_vertical"
        android:paddingLeft="10dip"
        android:singleLine="true"
        android:layout_weight="1"
        android:layout_margin="0dp"
        />  
``` 
### 2.需要修改这个方法中的时间戳格式  
NotePadProvider中的insert方法:
```
Long now = Long.valueOf(System.currentTimeMillis());
//修改 需要将毫秒数转换为时间的形式yy.MM.dd HH:mm:ss
Date date = new Date(now);
SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yy-MM-dd HH:mm:ss");
String dateFormat = simpleDateFormat.format(date);//转换为yy.MM.dd HH:mm:ss形式的时间
// If the values map doesn't contain the creation date, sets the value to the current time.
if (values.containsKey(NotePad.Notes.COLUMN_NAME_CREATE_DATE) == false) {
    values.put(NotePad.Notes.COLUMN_NAME_CREATE_DATE, dateFormat);
}

// If the values map doesn't contain the modification date, sets the value to the current
// time.
if (values.containsKey(NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE) == false) {
    values.put(NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE, dateFormat);
} 
```  
NoteEditor中的updateNote方法: 
```
long now = System.currentTimeMillis();
Date date = new Date(now);
SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yy-MM-dd HH:mm:ss");
String dateFormat = simpleDateFormat.format(date);
values.put(NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE, dateFormat);
``` 
### 3.NoteList的修改 
如果我们要加入时间戳，必须要将修改时间的列也投影出来.
所以我们将PROJECTION添加上修改时间： 
```
private static final String[] PROJECTION = new String[] {
            NotePad.Notes._ID, // 0
            NotePad.Notes.COLUMN_NAME_TITLE, // 1
            NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE,//添加修改时间
    };
``` 
PROJECTION只是定义了需要被取出来的数据列，而之后用Cursor进行数据库查询，再之后用Adapter进行装填。我们需要将显示列dataColumns和他们的viewIDs加入修改时间这一属性:
```
String[] dataColumns = { NotePad.Notes.COLUMN_NAME_TITLE, NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE }//加入修改时间;
int[] viewIDs = { android.R.id.text1, R.id.text2 }//加入修改时间;
``` 
## 搜索功能 
### 1.搜索组件在主页面的菜单选项中，所以应在list_options_menu.xml布局文件中添加搜索功能
```   
<item
      android:id="@+id/menu_search"
      android:icon="@android:drawable/ic_menu_search"
      android:title="@string/menu_search"
      android:showAsAction="always" />
```
### 2.新建一个查找笔记内容的布局文件note_search.xml  
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    >

    <SearchView
        android:id="@+id/search"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:iconifiedByDefault="false" />

    <ListView
        android:id="@+id/list"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

</LinearLayout>
```
### 3.在NoteList类中的onOptionsItemSelected方法中添加search查询的处理(跳转)
```

        case R.id.menu_search:  
        //查找功能  
        //startActivity(new Intent(Intent.ACTION_SEARCH, getIntent().getData()));  
          Intent intent = new Intent(this, NoteSearch.class);  
          this.startActivity(intent);  
          return true;  
```
### 4.新建一个NoteSearch类用于search功能的功能实现
```
package com.example.android.notepad;
import android.app.Activity;
import android.content.Intent;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;
import android.os.Bundle;
import android.widget.ListView;
import android.widget.SearchView;
import android.widget.SimpleCursorAdapter;
import android.widget.Toast;

public class NoteSearch extends Activity implements SearchView.OnQueryTextListener
{
    ListView listView;
    SQLiteDatabase sqLiteDatabase;
    /**
     * The columns needed by the cursor adapter
     */
    private static final String[] PROJECTION = new String[]{
            NotePad.Notes._ID, // 0
            NotePad.Notes.COLUMN_NAME_TITLE, // 1
            NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE//时间
    };

    public boolean onQueryTextSubmit(String query) {
        Toast.makeText(this, "您选择的是："+query, Toast.LENGTH_SHORT).show();
        return false;
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.note_search);
        SearchView searchView = (SearchView) findViewById(R.id.search);
        Intent intent = getIntent();
        if (intent.getData() == null) {
            intent.setData(NotePad.Notes.CONTENT_URI);
        }
        listView = (ListView) findViewById(R.id.list);
        sqLiteDatabase = new NotePadProvider.DatabaseHelper(this).getReadableDatabase();
        //设置该SearchView显示搜索按钮
        searchView.setSubmitButtonEnabled(true);

        //设置该SearchView内默认显示的提示文本
        searchView.setQueryHint("查找");
        searchView.setOnQueryTextListener(this);

    }
    public boolean onQueryTextChange(String string) {
        String selection1 = NotePad.Notes.COLUMN_NAME_TITLE+" like ? or "+NotePad.Notes.COLUMN_NAME_NOTE+" like ?";
        String[] selection2 = {"%"+string+"%","%"+string+"%"};
        Cursor cursor = sqLiteDatabase.query(
                NotePad.Notes.TABLE_NAME,
                PROJECTION, // The columns to return from the query
                selection1, // The columns for the where clause
                selection2, // The values for the where clause
                null,          // don't group the rows
                null,          // don't filter by row groups
                NotePad.Notes.DEFAULT_SORT_ORDER // The sort order
        );
        // The names of the cursor columns to display in the view, initialized to the title column
        String[] dataColumns = {
                NotePad.Notes.COLUMN_NAME_TITLE,
                NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE
        } ;
        // The view IDs that will display the cursor columns, initialized to the TextView in
        // noteslist_item.xml
        int[] viewIDs = {
                android.R.id.text1,
                android.R.id.text2
        };
        // Creates the backing adapter for the ListView.
        SimpleCursorAdapter adapter
                = new SimpleCursorAdapter(
                this,                             // The Context for the ListView
                R.layout.noteslist_item,         // Points to the XML for a list item
                cursor,                           // The cursor to get items from
                dataColumns,
                viewIDs
        );
        // Sets the ListView's adapter to be the cursor adapter that was just created.
        listView.setAdapter(adapter);
        return true;
    }
}
```
### 5.最后要在清单文件AndroidManifest.xml里面注册NoteSearch,否则无法实现界面的跳转 
```
<activity android:name=".NoteSearch" android:label="@string/menu_search" />
```  
## UI美化
### 1.先给NoteList换个主题，把黑色换成白色，在AndroidManifest.xml中NotesList的Activity中添加： 
```
android:theme="@android:style/Theme.Holo.Light"
```
### 2.创建数据库表地方添加颜色的字段： 
```
 @Override
    public void onCreate(SQLiteDatabase db) {
        db.execSQL("CREATE TABLE " + NotePad.Notes.TABLE_NAME + "   ("
        + NotePad.Notes._ID + " INTEGER PRIMARY KEY,"
        + NotePad.Notes.COLUMN_NAME_TITLE + " TEXT,"
        + NotePad.Notes.COLUMN_NAME_NOTE + " TEXT,"
        + NotePad.Notes.COLUMN_NAME_CREATE_DATE + " INTEGER,"
        + NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE + " INTEGER,"
        + NotePad.Notes.COLUMN_NAME_BACK_COLOR + " INTEGER" //颜色
        + ");");
       }
  ``` 

# 功能展示
1.打开APP，进入主菜单界面  
![image](https://user-images.githubusercontent.com/90608402/143822139-a735f973-30f7-45fd-b4ad-d91b1499e6b3.png)  
 
2.点击添加，添加备忘录  
![image](https://user-images.githubusercontent.com/90608402/143822235-f85a8553-dcd4-474a-a507-ac16264726e8.png)![image](https://user-images.githubusercontent.com/90608402/143822270-da757a7a-e04e-4dd6-9715-d800082e671b.png)   
 
3.输入内容，点击保存（红色框按钮）可以保存，删除（蓝色框按钮）可以删除  
![image](https://user-images.githubusercontent.com/90608402/143822448-e823a5ec-137b-4111-96a3-6dbc9fd69aff.png)![image](https://user-images.githubusercontent.com/90608402/143822681-43b11725-8a23-40a2-8e78-913a043434c1.png)
 
 4.在主菜单界面点击“查询笔记”按钮，进入笔记列表。（默认显示所有笔记，支持模糊查询） 
 
 ![image](https://user-images.githubusercontent.com/90608402/143823011-e4775982-6ca5-4516-b097-616c1c113c8d.png)![image](https://user-images.githubusercontent.com/90608402/143823200-dc5e5d32-6c9d-45e4-8c7e-c5c4ff37865b.png)
![image](https://user-images.githubusercontent.com/90608402/143823064-895f12f1-688f-4e04-9aa5-c87ccae41a24.png) 
