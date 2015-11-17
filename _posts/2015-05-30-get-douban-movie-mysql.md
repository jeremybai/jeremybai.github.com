---
layout: post
title: "python爬虫获取豆瓣电影——Python操作MySQL存储数据"
description: ""
categories: 
- python
- web
tags: [豆瓣,API]
---

　　爬虫抓到的数据需要存储到MySQL中，所以我们需要熟悉下使用Python操作MySQL数据库。首先你的机器上要安装MySQLdb，MySQLdb是用于Python连接Mysql数据库的接口，它实现了Python数据库 API规范V2.0，是基于MySQL C API上建立的。通过`import MySQLdb`语句查看是否安装MySQLdb。    
#1 连接数据库
　　连接数据库的前提在于首先你的数据库需要存在，假设你的机器上已经安装了MySQL，创建了数据库test_db，在test_db中有张名为test_info的表，`PS：记得将你的<password>替换为你的密码`：  
{% highlight python %}
#!/usr/bin/python
# -*- coding: UTF-8 -*-
import MySQLdb

def run():
    try:
        # 打开数据库连接，设置charset为utf8，否则存入数据库时在workbench显示为乱码
        db = MySQLdb.connect("localhost", "root", <password>, "test_DB", charset='utf8')
    except MySQLdb.Error, e:
        print("Mysql Connect Error %d: %s" % (e.args[0], e.args[1]))
        return
    # 使用cursor()方法获取操作游标
    cursor = db.cursor()
    # 使用execute方法执行SQL语句
    cursor.execute("SELECT VERSION()")
    # 使用 fetchone() 方法获取一条数据库。
    data = cursor.fetchone()
    print "Database version : %s " % data
    try:
        # 关闭数据库连接
        db.close()
    except MySQLdb.Error, e:
        print("Mysql Close Error %d: %s" % (e.args[0], e.args[1]))
        return

if __name__ == '__main__':
    run()
{% endhighlight %} 
#2 创建数据库
　　如果数据库连接存在我们可以使用execute()方法来为数据库创建表，如下所示创建表test_info，我只列出了run函数的内容，你用它替换之前的代码就可以了：   
{% highlight python %}
def run():
    try:
        # 打开数据库连接，设置charset为utf8，否则存入数据库时在workbench显示为乱码
        db = MySQLdb.connect("localhost", "root", <password>, "test_DB", charset='utf8')
    except MySQLdb.Error, e:
        print("Mysql Connect Error %d: %s" % (e.args[0], e.args[1]))
        return
    # 使用cursor()方法获取操作游标
    cursor = db.cursor()
    # 如果数据表已经存在使用 execute() 方法删除表。
    cursor.execute("DROP TABLE IF EXISTS test_info")
    # 创建数据表SQL语句
    sql = """CREATE TABLE test_info (
              test_info_id INT NOT NULL AUTO_INCREMENT COMMENT '',
              movie_name VARCHAR(200) NULL COMMENT '',
              movie_year VARCHAR(45) NULL COMMENT '',
              movie_type VARCHAR(45) NULL COMMENT '',
              PRIMARY KEY (test_info_id)  COMMENT '')
            ENGINE = InnoDB
            DEFAULT CHARACTER SET = utf8;"""
    # 使用execute方法执行SQL语句
    cursor.execute(sql)
    try:
        # 关闭数据库连接
        db.close()
    except MySQLdb.Error, e:
        print("Mysql Close Error %d: %s" % (e.args[0], e.args[1]))
        return
{% endhighlight %} 
#3 插入数据库
　　下面展示下如何向数据库中插入数据：  
{% highlight python %}
def run():
    try:
        # 打开数据库连接，设置charset为utf8，否则存入数据库时在workbench显示为乱码
        db = MySQLdb.connect("localhost", "root", <password>, "test_DB", charset='utf8')
    except MySQLdb.Error, e:
        print("Mysql Connect Error %d: %s" % (e.args[0], e.args[1]))
        return
    # 使用cursor()方法获取操作游标
    cursor = db.cursor()
    # 创建数据表SQL语句
    sql = """INSERT INTO test_info(movie_name,
         movie_year, movie_type)
         VALUES ('盗梦空间', '2010', '剧情')"""
    try:
        # 使用execute方法执行SQL语句
        cursor.execute(sql)
        # 提交到数据库执行
        db.commit()
    except MySQLdb.Error, e:
        # 出错时回滚
        print("Mysql Execute Error %d: %s" % (e.args[0], e.args[1]))
        db.rollback()
    try:
        # 关闭数据库连接
        db.close()
    except MySQLdb.Error, e:
        print("Mysql Close Error %d: %s" % (e.args[0], e.args[1]))
        return
{% endhighlight %}
#4 查询数据库
　　下面展示了获取并打印出数据库中的movie_name字段。   
{% highlight python %}
def run():
    try:
        # 打开数据库连接，设置charset为utf8，否则存入数据库时在workbench显示为乱码
        db = MySQLdb.connect("localhost", "root", <password>, "test_DB", charset='utf8')
    except MySQLdb.Error, e:
        print("Mysql Connect Error %d: %s" % (e.args[0], e.args[1]))
        return
    # 使用cursor()方法获取操作游标
    cursor = db.cursor()
    # 创建数据表SQL语句
    sql = "SELECT movie_name FROM test_info"
    try:
        cursor.execute(sql)
    except MySQLdb.Error, e:
        # 出错时回滚
        print("Mysql Execute Error %d: %s" % (e.args[0], e.args[1]))
        db.rollback()
    # 获取所有记录列表
    results = cursor.fetchall()
    for item in results:
        print str(item).decode("unicode_escape")
    try:
        # 关闭数据库连接
        db.close()
    except MySQLdb.Error, e:
        print("Mysql Close Error %d: %s" % (e.args[0], e.args[1]))
        return
{% endhighlight %}
#5 更新数据库
　　除了上面的操作，更新数据库中的数据也是一项常见的操作，下面的代码展示了如何将电影信息中《盗梦空间》的年份改为2015，其实操作是类似的，只需要修改下sql语句即可。  
{% highlight python %}
def run():
    try:
        # 打开数据库连接，设置charset为utf8，否则存入数据库时在workbench显示为乱码
        db = MySQLdb.connect("localhost", "root", <password>, "test_DB", charset='utf8')
    except MySQLdb.Error, e:
        print("Mysql Connect Error %d: %s" % (e.args[0], e.args[1]))
        return
    # 使用cursor()方法获取操作游标
    cursor = db.cursor()
    # 创建数据表SQL语句
    sql = "UPDATE test_info SET movie_year = '2015' WHERE movie_name = '%s'" % (MySQLdb.escape_string('盗梦空间'))
    try:
        # 执行SQL语句
        cursor.execute(sql)
        # 提交到数据库执行
        db.commit()
    except MySQLdb.Error, e:
        # 出错时回滚
        print("Mysql Execute Error %d: %s" % (e.args[0], e.args[1]))
        db.rollback()
    try:
        # 关闭数据库连接
        db.close()
    except MySQLdb.Error, e:
        print("Mysql Close Error %d: %s" % (e.args[0], e.args[1]))
        return
{% endhighlight %}

----------

　　如果在你的程序中遇到下面的错误：  

	Mysql Execute Error 1064: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near XXX
　　这个错误多半是因为你没有对你的sql语句中的特殊字符进行转义。这个错误你没有线索可以去定位错误，你可以通过检查下面列出的问题来查看是否在你的sql语句中出现：  

	1 是否遗漏或者包含了不必要的字符：`!@#$%^&*()-_=+[]{}\|;:'",<>/?`  
	2 是否将关键字遗漏或者放在了错误的位置：select, into或者其他。  
	3 在你的sql语句中yo看起来像ascii字符的unicode字符。  
	4 在关键字之间有遗漏或者放错位置的空格或者换行。  
	5 不匹配的单引号、双引号、小括号以及大括号等等。  
　　如果仔细检查之后没有发现上述问题，那么有可能是你在将参数传入sql语句时，这个参数中带了某些特殊字符，此时可以使用MySQLdb自带的针对MySQL的字符转义函数`escape_string`将参数进行转义。在这个项目中我就遇到了电影的描述信息中存在单引号等特殊字符造成出现上述的错误，通过在构造sql语句时将参数进行转义之后问题就解决了。    
　　cursor.execute()函数可以接受一个参数，也可以接受两个参数，这两种情况如下所示：  
　　(1) 两个参数    
{% highlight python %}
cursor.execute("UPDATE test_info SET movie_year = %s WHERE movie_name = %s", ('''20'1(6''', '盗梦空间'))
{% endhighlight %}
　　这种格式是接受两个参数，MySQLdb会自动替你对字符串进行转义和加引号，不必再自己进行转义，执行完此语句之后，表中盗梦空间这条记录的时间字段被修改为`20'1(6`。    
　　(2) 一个参数  
　　这种格式是利用字符串格式化生成sql语句，也就是传给execute一个参数：   
{% highlight python %}
cursor.execute("UPDATE test_info SET movie_year = %s WHERE movie_name = %s" % ('''20'1(6''', '盗梦空间'))
{% endhighlight %}
　　运行时就会出现错误：   

	Mysql Execute Error 1064: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''1(6 WHERE movie_name = 盗梦空间' at line 17
　　这种格式需要自己对字符串转义和增加引号，应该修改为：  
{% highlight python %}
cursor.execute("UPDATE test_info SET movie_year = '%s' WHERE movie_name = '%s'" % (MySQLdb.escape_string('''20'1(6'''), '盗梦空间'))
{% endhighlight %}
　　改完之后记录就被正常更新了，**需要注意，在第二种方法中`%s`需要加上单引号。** 相比较而言，还是第一种方法比较好用。  
