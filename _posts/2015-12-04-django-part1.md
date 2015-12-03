---
layout: post
title: django install
tags:  [django]
categories: [django]
author: mingtian
excerpt: "django"
---

### django install

 * 查看版本
 
 ~~~
   import django
   print(django.get_version())
 ~~~
 
 * 启动服务器
 
 > python manage.py runserver 0.0.0.0:8000
 
 * 创建 project
 
 > django-admin startproject mysite
 
 * 创建 app
  
  > python manage.py startapp polls 
  
 * models 对应SQL

 > python manage.py sqlmigrate polls 0001
 
 
 * 应用migrate
 
 > python manage.py migrate
 
 命令行
 
 > django shell: python manage.py shell
 
 
 ### django models  
 
 #### 定义 model  
 
 
 ~~~
 
 class Author(models.Model):
    class Meta:
         verbose_name_plural='作者信息' # 显示在菜单中的名称 admin
         db_table = 'books_author' # 对应数据库中的表名

    user = models.OneToOneField(User,verbose_name="用户名")  # 系统用户 admin

    # 没有指定 verbose_name, 在页面中显示的是 首字母大写
    blog = models.CharField("博客地址",max_length=128, blank=True)

    location = models.CharField("位置信息",max_length=128, blank=True)

    occupation = models.CharField(max_length=64, blank=True)

    reward = models.IntegerField(default=0, blank=True)

    topic_count = models.IntegerField(default=0, blank=True)

    post_count = models.IntegerField(default=0, blank=True)


    def last_login(self): # 显示关联数据库字段
        return self.user.last_login
    last_login.short_description='上次登录时间'


    @staticmethod
    def begin_save():
        print "begin save"

    @staticmethod
    def end_save():
        print "end save"

    def save(self, *args, **kwargs): # 重写保存方法
        self.begin_save()
        super(Author, self).save(*args, **kwargs) # Call the "real" save() method.
        self.end_save()

    def __str__(self):
        return self.user.username
        
        
  class Book(models.Model):
    class Meta:
        verbose_name_plural='书籍'

    title = models.CharField(max_length=100, help_text='书籍名称')
    authors = models.ManyToManyField(Author)
    publisher = models.ForeignKey(Publisher)
    price = models.FloatField()
    publication_date = models.DateField()

    def __str__(self):
        return self.title
        
 ~~~
 

 #### 查询
 
 * Book.objects.all()
 * Book.objects.filter(title='book1')
 * Book.objects.all().filter(title__startswith='book1')
 * Book.objects.all().filter(title__constains='book1')
 * Book.objects.get(Q(publisher_name='pub1'))
 * Book.objects.filter(Q(publisher__name='pub1')|Q(publisher__name='pub2'))


*django making queries*  
<https://docs.djangoproject.com/en/1.9/topics/db/queries>

#### Managers


 