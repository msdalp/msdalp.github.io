---
layout: post
title:  "Java Sqlite Fast Insert with PreparedStatement"
date:   2014-12-11 19:00:00
categories:
---
In java you can use two ways of inserting data into your table. First one is using classical transactions and the second one is prepared statements.


{% highlight java %}

public synchronized boolean insertTfStats( List<RawDataObject> playerObj) {
        final Transaction trans = this.getWritableDatabase();
        boolean retval = false;

       
        try { 

            for (RawDataObject obj : playerObj) {

                HashMap<String, Object> values = new HashMap<>(); 
                values.put("TIMESTAMP", obj.timestamp);
                values.put("HALF", obj.half);
                values.put("MIN", obj.min);
                values.put("SEC", obj.sec);
                values.put("OBJECT_TYPE", obj.type);
                values.put("OBJECT_ID", obj.objectId); 
                values.put("X_POS", obj.x);
                values.put("Y_POS", obj.y);
                values.put("Z_POS", obj.z);
                trans.insert(SQL.TABLE_STAT, null, values);

            }

            retval = true;
        } catch (Exception ex) {
            logger.error("Cannot add  stat data", ex);
        } finally {
            trans.close();
        }
        return retval;
    } 

{% endhighlight %}
Doing the same insert with prepared statement.
{% highlight java %} 
public synchronized boolean InsertTfStats(List<RawDataObject> playerObj) {
        final Transaction trans = this.getWritableDatabase();
        boolean retval = false;

        try {
            
            PreparedStatement stmt = trans.preparedStmt("INSERT INTO local_stat VALUES (?,?,?,?,?,?,?,?,?)");
       
            for (RawDataObject obj : playerObj) {
                stmt.setObject(1, obj.timestamp);
                stmt.setObject(2, obj.half);
                stmt.setObject(3, obj.min);
                stmt.setObject(4, obj.sec);
                stmt.setObject(5, obj.type);
                stmt.setObject(6, obj.objectId);
                stmt.setObject(7, obj.x);
                stmt.setObject(8, obj.y);
                stmt.setObject(9, obj.z);
                stmt.addBatch();
            }
            stmt.executeBatch(); 
            retval = true;
        } catch (Exception ex) {
            logger.error("Cannot add stat data", ex);
        } finally {
            trans.close();
        }
        return retval;
    }
{% endhighlight %}
