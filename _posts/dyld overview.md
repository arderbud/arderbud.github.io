## dyld overview

I just try to overview the dyld for that developers who has difficult to read dyld opensource. Let's start.

####  Background

For iOS app developers you should know that when we write an app,we need  normally depend on a lots of dynamic frameworks.Those framework do not packeted in our app, they are in system directory some where.How our code can find them, use them? This is the work that dyld should to do.

#### User space

When we tap an app icon,then an app is started.App is a process.It has owned user space . 

What is user space? If you don't know  clearly,you can just consider it as data address. Assume code `int i =3; int *p = &i`in c langurage ,p is the address of var i.

The code where  we write app to,and the dynamic framework we denpening on,They are all in the user space.Like figure 1-1

Figure 1-1

<img src="https://s2.ax1x.com/2020/01/03/larzGj.png" alt="larzGj.png" style="zoom:50%;" />

You can set `DYLD_PRINT_SEGMENTS` environment variables to see dyld logs in case deepen the impression. Also user `image list`lldb instrument  to debug.

#### Symbol bind

Now,we have knowed  all the addresses of dynamic framework we depend on.Then we can bind symbol.

What is symbol bind?For example, assume  code `Class cls = [NSObjcet class];`,we use the `+[NSObject class]`method.When we write code, we donnot know the the address of `+[NSObject class]`method.We can use a placeholer for the method, like figure 1-2.

Figure 1-2.

<img src="https://s2.ax1x.com/2020/01/03/larxiQ.png" alt="larxiQ.png" style="zoom: 50%;" />

But now we know the actual address,the we can fill in it.Assume the address of method `+[NSObject class]` is 0x7fffad1bf140,After fill in,like figure 1-3

Figure 1-3

<img src="https://s2.ax1x.com/2020/01/03/layKhQ.png" alt="layKhQ.png"  />

This is so-called `bind`.

**So far,the dyld's main work is done.**

#### One more thing

If you read the dyld opensource,you can't go around the `dyld_shared_cache`，`imageLoader`,what are they? I just apply a figure 1-4,leave the rest to you.

Figure 1-4.

![laTAPI.png](https://s2.ax1x.com/2020/01/03/laTAPI.png)





