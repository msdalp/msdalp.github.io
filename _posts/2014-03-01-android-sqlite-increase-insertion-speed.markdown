---
layout: post
title:  "Increase Insertion Speed on Android SQLite"
date:   2014-03-04 16:40:24
categories:
---
I've been doing some insert operations on Android with 3000 rows to 15000 rows.A service was planned to update for every one hour and insert all over again.The problem with that it was so slow.Inserting 3000 rows took 150 seconds and there was many tables with that kind of data it will not be able to insert even in one hour.I was using the usual way to do it with ContentValues.Please be aware it's a set function so I deleted the datas before inserting anything.

{% highlight java %}
public synchronized boolean setData(List<YourObject> _objects) {
        final SQLiteDatabase db = this.getWritableDatabase();
        boolean retval = false;
        try {
            db.delete("table_data", null, null);
            for (YourObject object : _objects) {
                ContentValues contentValues = new ContentValues();
                contentValues.put("ID", object.id);
                contentValues.put("NAME", object.name);
                contentValues.put("ABBR", object.abbr);
                contentValues.put("FOUND_YEAR", object.foundYear);
                db.insert("table_data", null, contentValues);
            }
            retval = true;
        } catch (Exception ex) {
            Log.e(TAG, "setData exception", ex);
        } finally {
            db.close();
        }
        return 
}
{% endhighlight %}
After some digging it was obvious for this kind of task using SqliteStatements is much better option.You can check the details of SqliteStatement here. Combining transactions with compiled statements yielded a tremendous performance boost: From the original 150 seconds on the Android device (Asus TF300) down to an incredible 1.5 seconds.

{% highlight java %}
public synchronized boolean setData(List<YourObject> _objects) {
        final SQLiteDatabase db = this.getWritableDatabase();
        final String INSERT_DATA = "INSERT INTO table_data VALUES (?,?,?,?);";
        boolean retval = false;
        try {
            db.delete("table_data", null, null);
            SQLiteStatement statement = db.compileStatement(INSERT_DATA);
            db.beginTransaction();
            for (YourObject object : _objects) {
                statement.clearBindings();
                statement.bindLong(1, object.id);
                statement.bindString(2, object.name);
                statement.bindString(3, object.abbr);
                statement.bindLong(4, object.foundYear);
                statement.execute();
            }
            db.setTransactionSuccessful();
            db.endTransaction();
            retval = true;
        } catch (Exception ex) {
            Log.e(TAG, "setData exception", ex);
        } finally {
            db.close();
        }
        return retval;
}
{% endhighlight %}
