---
layout: page
section: Code
category: code
date: 2013-04-09
title: "dotnet拾遗"
description: "dotnet拾遗"
summary: "dotnet拾遗"
---
 
获取文件绝对路径
---------------

    System.IO.Path.Combine(HttpRuntime.AppDomainAppPath,"App_Data\\test.txt");

字符连接
-------------

<a href="http://blog.zhaojie.me/2009/12/valuable-posts-index.html#string-concat-perf" target="blank">重谈字符串连接性能</a>

StringBuilder：如果能够确定目标字符串的最终长度，则可以使用StringBuilder。如果不能确定的话，也可以在一开始指定更大的容量，减少扩容的次数。

    var builder = new StringBuilder(count * STR.Length);

String.Concat：如果不能确定最终长度，但是能够确定字符串的个数（如这个场景），可以将它们放在一个数组中，并调用String.Concat进行连接。
    
    String.Concat(array);

EF下修改指定的字段
--------------------

    //NorthwindEntities : ObjectContext
    NorthwindEntities entities = new NorthwindEntities();

    entities.Products.Attach(product);
    var entry = entities.ObjectStateManager.GetObjectStateEntry(product);
    entry.SetModifiedProperty("ProductName"); 
    entry.SetModifiedProperty("Discontinued");
    entities.SaveChanges();

另外一种（继承自DBContext）修改方式：

    public void UpdateObject(T obj,params string[] propertys)
    {
        var entry = context.Entry<T>(obj);

        if (propertys!=null)
        {
            IObjectContextAdapter objectContextAdatper = context;
            ObjectContext objectContext = objectContextAdatper.ObjectContext;
            ObjectStateEntry ose = objectContext.ObjectStateManager.GetObjectStateEntry(obj);
            foreach (string property in propertys)
            {
                ose.SetModifiedProperty(property);
            }
        }

        if (entry.State == EntityState.Detached)
        {
            entry.State = EntityState.Modified;
        }
        context.SaveChanges();
    }

EF使用注意事项
--------------

    //这样才会在数据库中分页.
    query.Skip((pageInfo.PageIndex - 1) * pageInfo.PageSize).Take(pageInfo.PageSize).ToList();

    //尽量避免延迟加载，而是预加载
    var list = db.People.Include("StudentGrades").Where(ent => ent.PersonID < 30).ToList();


