---
layout: post
title:  "Android Sqlite Example"
date:   2014-03-04 16:40:24
categories:
---
SQLite is embedded into every Android device. Using an SQLite database in Android does not require a setup procedure or administration of the database. You only have to define the SQL statements for creating and updating the database. Afterwards the database is automatically managed for you by the Android platform.

If your application creates a database, this database is by default saved in the directory DATA/data/APP\_NAME/databases/FILENAME. The parts of the above directory are constructed based on the following rules. DATA is the path which the Environment.getDataDirectory() method returns. APP\_NAME is your application name. FILENAME is the name you specify in your application code for the database.

To create and upgrade a database you have to create a subclass of the SQLiteOpenHelper so you have to extend SQLiteOpenHelper when you define a database class.Here the simplest example how you create a database named "mydb" and a table in it as "data".You have to override onCreate() and onUpgrade() methods.


{% highlight java %}
  public class MyDb extends SQLiteOpenHelper {
 
    public static final String TAG = "MyDb";
    public static final int VERSION = 1;
    public static final String DATABASENAME = "mydb";
    public MyDb(Context context) {
        super(context, DATABASENAME, null, VERSION);
    }
 
    @Override
    public void onCreate(SQLiteDatabase db) {
        db.execSQL("CREATE TABLE IF NOT EXISTS data" +
            + " (ID INTEGER PRIMARY KEY, "
            + "DATA_ID TEXT, "
            + "VALUE TEXT);");
    }
 
    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        
    }
}
{% endhighlight %}

The SQLiteOpenHelper class provides the getReadableDatabase() and getWriteableDatabase() methods to get access to an SQLiteDatabase object; either in read or write mode.Also SQLiteDatabase provides insert(), update(), delete(), and query methods.To make a query you can use either of rawquery() or query().Also keep this mind that you should always make sure you use methods as synchronized.Here another example of adding data into database and retrieving from it.

We want to use synchronized for every database operation method to avoid from getting database locked errors. I used a list of objects which are "Data" objects that make things more readable but you can use just an object or even use only the areas like objects that make things more readable but you can use just an object or even use only the areas like (long \_id, long \_dataId, String \_value).All the database operations are done in try-catch-finally block and returning a boolean value to show if the insertion was successful or not is the convenient way to do it. ContentValues is the class given by Android and its required for any database operation.

REMARK:You will see a commented out code which is db.insertWithOnConflict method that is a very good way to insert with insertion algorithms at the end. I used SQLiteDatabase.CONFLICT\_REPLACE which basicly updates the old one with the new one if it's already in the database. And after the all operations are done always remember to close database otherwise you will get errors saying that database is already opened.
{% highlight java %}
public synchronized boolean addData(List<YourObject> _objects) {
        final SQLiteDatabase db = this.getWritableDatabase();
        boolean retval = false;
        try {
            for (YourObject object : _objects) {
                ContentValues contentValues = new ContentValues();
                contentValues.put("ID", object.id);
                contentValues.put("DATA_ID", object.dataId);
                contentValues.put("VALUE", object.value);
                db.insert("data",null,contentValues);
//db.insertWithOnConflict("data", null, contentValues, SQLiteDatabase.CONFLICT_REPLACE);
            }
            retval = true;
        } catch (Exception ex) {
            Log.e(TAG, "addData exception", ex);
        } finally {
            db.close();
        }
        return retval;
}
{% endhighlight %}
Queries can be created via the rawQuery() and query() methods or via the SQLiteQueryBuilder class . rawQuery() directly accepts an SQL select statement as input. query() provides a structured interface for specifying the SQL query. SQLiteQueryBuilder is a convenience class that helps to build SQL queries.
{% highlight java %}
public synchronized String getDataValue(long _id) {
        final SQLiteDatabase db = this.getReadableDatabase();
        String retval = null;
        try {
            Cursor cursor = db.query("data", new String[]{"VALUE"},
                    "ID = ?", new String[]{String.valueOf(_id)}, null, null, null);
            if (cursor != null) {
                if (cursor.moveToFirst()) {
                    do {
                        retval = cursor.getString(0);
                    } while (cursor.moveToNext());
                }
            }
        } catch (Exception ex) {
            retval = null;
            Log.e(TAG, "Cannot get data value:", ex);
        } finally {
            db.close();
        }
        return retval;
}
{% endhighlight %}
The following gives an example of a rawQuery() and a query() call.
{% highlight java %}
Cursor cursor = getReadableDatabase().
  rawQuery("select * from data where ID = ?", new String[] { _id });
 
database.query("data", 
  new String[] { ID, DATA_ID, VALUE }, 
  null, null, null, null, null);
{% endhighlight %}
Parameters of the query() method

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;border-color:#aaa;}
.tg td{font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#aaa;color:#333;background-color:#fff;}
.tg th{font-family:Arial, sans-serif;font-size:14px;font-weight:normal;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#aaa;color:#fff;background-color:#f38630;}
</style>
<table class="tg">
  <tr>
    <th class="tg-031e">Parameters</th>
    <th class="tg-031e">Values</th>
  </tr>
  <tr>
    <td class="tg-031e">String dbName</td>
    <td class="tg-031e">The table name to compile the query against.</td>
  </tr>
  <tr>
    <td class="tg-031e">String[] columnNames</td>
    <td class="tg-031e">A list of which table columns to return. Passing "null" will return all columns.</td>
  </tr>
  <tr>
    <td class="tg-031e">String whereClause</td>
    <td class="tg-031e">Where-clause, i.e. filter for the selection of data, null will select all data.</td>
  </tr>
  <tr>
    <td class="tg-031e">String[] selectionArgs</td>
    <td class="tg-031e">You may include ?s in the "whereClause"". These placeholders will get replaced by the values
 from the selectionArgs array.</td>
  </tr>
  <tr>
    <td class="tg-031e">String[] groupBy</td>
    <td class="tg-031e">A filter declaring how to group rows, null will cause the rows to not be grouped.</td>
  </tr>
  <tr>
    <td class="tg-031e">String[] having</td>
    <td class="tg-031e">Filter for the groups, null means no filter.</td>
  </tr>
  <tr>
    <td class="tg-031e">String[] orderBy</td>
    <td class="tg-031e">Table columns which will be used to order the data, null means no ordering.</td>
  </tr>
</table>
After you have all database create, upgrade, add, and a get method how you will call it from the activity?Here is the how your main activity look like.
{% highlight java %}
package com.test.db;
 
import android.app.Activity;
import android.os.Bundle;
import android.util.Log;
import java.util.LinkedList;
import java.util.List;
 
public class MainActivity extends Activity {
 
    public static String TAG = "MyDbActivity";
    public static MyDb MyDb;
 
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
        MyDb = new MyDb(this);
        List<Data> data = new LinkedList<Data>();
        data.add(new Data(1, 2032, "data1"));
        if (MyDb.addData(data)) {
            Log.d("Data value is:", MyDb.getDataValue(1L));
        }
    }
 
    @Override
    protected void onDestroy() {
        super.onDestroy();
        MyDb.close();
        MyDb = null;
    }
 
    public class Data {
 
        public long id;
        public long dataId;
        public String value;
 
        public Data() {
        }
 
        public Data(long _id, long _dataId, String _value) {
            this.id = _id;
            this.dataId = _dataId;
            this.value = _value;
        }
    }
}
{% endhighlight %}
You can download both mainactivity and db class from the links given below:
MainActivity:<a href="https://gist.github.com/msdalp/9297781">Github Link</a>
Db Class: <a href="https://gist.github.com/msdalp/9297983">Github Link</a>

