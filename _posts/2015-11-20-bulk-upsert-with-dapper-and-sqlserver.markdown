---
layout: post
title:  "Bulk Upsert with Dapper and Sql Server"
date:   2015-11-20 22:57:00 -0500
categories: c# sql
---
In this post, I will discuss a method that I use to perform bulk upsert (insert or update) of data into sql server.  I will be using c# and the awesome [dapper][dapper] library.  This is useful when doing an import from a csv or excel file into a database table.  I might, at some point in the future, show some code for the actual file parsing.  For now, you can find all of the code used in this post [here][github_source].

For this example, I'll use the table defined below.  

{% highlight sql %}
CREATE TABLE [dbo].[sometable](
  [sometable_id] [bigint] IDENTITY(1,1) NOT NULL,
  [unique_field] [int] null,
  [field1] [int] null,
  [field2] [varchar](max) null,
  [field3] [bit] null,
 CONSTRAINT [pk_sometable] PRIMARY KEY ([sometable_id] ASC)
)
{% endhighlight %} 

Since I'll be using sql table-valued parameters we'll need to create a corresponding custom data type with all of the fields to be imported.

{% highlight sql %}
CREATE TYPE [dbo].[sometable_type] AS TABLE(
  [unique_field] [int] null,
  [field1] [int] null,
  [field2] [varchar](max) null,
  [field3] [bit] null
)
{% endhighlight %} 

Next I'll create a stored procedure that takes a parameter of the above type and uses the merge command to either insert the data or update it if it already exists.  I am using `unique_field` to determine equality between rows in the table.  You could use one or multiple external fields to determine uniqueness based on business rules.  You could use the primary key for this, but many of my use cases involve the same external data being imported multiple times.  In order to use the primary key, it would have to be attached to the data in subsequent calls.

{% highlight sql %}
create PROCEDURE [dbo].[sometable_upsert] (
  @data [dbo].[sometable_type] READONLY
)
AS
DECLARE @T TABLE([id] int, [_rownumber] int)

MERGE INTO [dbo].[sometable] AS t
USING (SELECT *, [_rownumber] = ROW_NUMBER() OVER (ORDER BY (SELECT 1)) FROM @data) AS s
ON
(
  t.[unique_field] = s.[unique_field]
)
WHEN MATCHED THEN UPDATE SET
  t.[field1] = s.[field1],
  t.[field2] = s.[field2],
  t.[field3] = s.[field3]

WHEN NOT MATCHED BY TARGET THEN INSERT
(
  [unique_field],
  [field1],
  [field2],
  [field3]
)
VALUES
(
  s.[unique_field],
  s.[field1],
  s.[field2],
  s.[field3]
)
OUTPUT Inserted.[sometable_id], s.[_rownumber] INTO @T ;
SELECT [id] FROM @T ORDER BY [_rownumber]
{% endhighlight %} 

Now moving out of the database to the application side of things, I'll create a simple poco class in c# that we'll use to map to the database.

{% highlight c# %}
public class SomeTable
{
    public int unique_field { get; set; }
    public int field1 { get; set; }
    public string field2 { get; set; }
    public bool field3 { get; set; }
} 
{% endhighlight %} 

I will now use [dapper][dapper] to call the stored procedure passing in a list of `SomeTable` objects as my table-valued parameter.

{% highlight c# %}
public class Repository
{
    public static List<int> StoreData(List<SomeTable> list)
    {
        string connectionString = ConfigurationManager.ConnectionStrings["mydb"].ConnectionString;
        List<int> ids = new List<int>();
        using (var connection = new SqlConnection(connectionString))
        {
            ids = connection.Query<int>("sometable_upsert",
            new
            {
                data = list.AsTableValuedParameter("dbo.sometable_type")
            },
            commandType: CommandType.StoredProcedure).ToList();
        }
        return ids;
    }
}
{% endhighlight %}

The extension method `AsTableValuedParameter` is a helper function that we use to map the table into the stored procedure parameter.  It is a little long, but only needs to be defined once.  You can then use it for many different repositories and stored procs.

{% highlight c# %}
public static class DapperExtension
{
    public static SqlMapper.ICustomQueryParameter AsTableValuedParameter<T>(
        this IEnumerable<T> enumerable,
        string typeName,
        IEnumerable<string> orderedColumnNames = null)
    {
        return enumerable.AsDataTable(orderedColumnNames).AsTableValuedParameter(typeName);
    }
}

public static class EnumerableExtension
{
    public static DataTable AsDataTable<T>(this IEnumerable<T> enumerable, IEnumerable<string> orderedColumnNames = null)
    {
        var dataTable = new DataTable();
        if (typeof(T).IsValueType)
        {
            dataTable.Columns.Add("NONAME", typeof(T));
            foreach (T obj in enumerable)
            {
                dataTable.Rows.Add(obj);
            }
        }
        else
        {
            PropertyInfo[] properties = typeof(T).GetProperties(BindingFlags.Public | BindingFlags.Instance);
            PropertyInfo[] readableProperties = properties.Where(w => w.CanRead).ToArray();
            var columnNames = (orderedColumnNames ?? readableProperties.Select(s => s.Name)).ToArray();
            foreach (string name in columnNames)
            {
                dataTable.Columns.Add(name, readableProperties.Single(s => s.Name.Equals(name)).PropertyType);
            }

            foreach (T obj in enumerable)
            {
                dataTable.Rows.Add(
                    columnNames.Select(s => readableProperties.Single(s2 => s2.Name.Equals(s)).GetValue(obj))
                        .ToArray());
            }
        }
        return dataTable;
    }
}
{% endhighlight %}

In closing, I hope that this helps the two people (myself included) who might actually read it. If are the other person, please feel free to contact me at the email below.  You can yell at me for all of the stupid things I probably did wrong or, even better, show me a better way. 


[github_source]: https://github.com/jpschroeder/bulkupsertsample
[dapper]: https://github.com/StackExchange/dapper-dot-net