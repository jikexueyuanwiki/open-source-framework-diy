# 缓存相关代码的演变

## 问题引入

上次我参与某个大型项目的优化工作，由于系统要求有比较高的TPS，因此就免不了要使用缓冲。

该项目中用的缓冲比较多，有MemCache，有Redis，有的还需要提供二级缓冲，也就是说应用服务器这层也可以设置一些缓冲。

当然去看相关实现代代码的时候，大致是下面的样子。

```
public void saveSomeObject(SomeObject someObject){  
  
    MemCacheUtil.put("SomeObject",someObject.getId(),someObject);  
  
    //下面是真实保存对象的代码  
  
   
  
}  
  
public SomeObject getSomeObject(String id){  
  
    SomeObject someObject = MemCacheUtil.get("SomeObject",id);  
  
    if(someObject!=null){  
  
         someObject=//真实的获取对象  
  
         MemCacheUtil.put("SomeObject",someObject.getId(),someObject);  
  
    }  
  
    return someObject;  
  
}  
```
 

很明显与缓冲相关的代码全部是耦合到原来的业务代码当中去的。

后来由于MemCache表现不够稳定，而且MemCache的功能，也可以由Redis完全进行实现，于是就决定从系统中取消MemCache，换成Redis的实现方案，于是就改成如下的样子：

```
public void saveSomeObject(SomeObject someObject){  
  
    RedisUtil.put("SomeObject",someObject.getId(),someObject);  
  
    //下面是真实保存对象的代码  
  
   
  
}  
  
public SomeObject getSomeObject(String id){  
  
    SomeObject someObject = RedisUtil.get("SomeObject",id);  
  
    if(someObject!=null){  
  
         someObject=//真实的获取对象 <span></span>RedisUtil.put("SomeObject",someObject.getId(),someObject);  
  
    }  
  
    return someObject;  
  
}  
```
 

这一通改下来，开发人员已经晕头晕脑的了，后来感觉性能还是不够高，这个时候，要把一些数据增加二级缓冲，也就是说，本地缓冲有就取本地，本地没有就取远程缓冲  

于是，上面的代码又是一通改，变成下面这个样子：

```
public void saveSomeObject(SomeObject someObject){  
  
    LocalCacheUtil.put("SomeObject",someObject.getId(),someObject);  
  
    RedisUtil.put("SomeObject",someObject.getId(),someObject);  
  
    //下面是真实保存对象的代码  
  
   
  
}  
  
public SomeObject getSomeObject(String id){  
  
    SomeObject someObject = LocalCacheUtil.get("SomeObject",id);  
  
    if(someObject!=null){  
  
        return someObject;  
  
    }  
  
    someObject = RedisUtil.get("SomeObject",id);  
  
    if(someObject!=null){  
  
         someObject=//真实的获取对象   
  
         RedisUtil.put("SomeObject",someObject.getId(),someObject);  
  
    }  
  
    return someObject;  
  
}  
```
 

但是这个时候就出现一个问题： 

由于在某一时刻修改值的只能是某一台计算机，这个时候，其它的计算机的本地缓冲实际上与远程及数据库中的数据会不一致，这个时候，可以有两种办法实现，一种是利用Redis的请阅发布机制进行数据同步，这种方式，会保证数据能够被及时同步。

另外一种方法就是设置本地缓冲的有效时间比较短，这样，允许在比较短的时间段内出现数据不一致的情况。

不管怎么样，功能是实现了，程序员小伙伴这个时候已经改得眼睛发黑，手指发麻，几乎接近崩溃了。

很明显这种实现方式是不好的，于是项目组又提出了改进意见，能否采用注解方式进行标注，让程序员只要声明就可以？Good idea，于是，又变成了下面的样子：

```
@Cache(type="SomeObject",parameter="someObject",key="${someObject.id}")  
  
public void saveSomeObject(SomeObject someObject){  
  
    //下面是真实保存对象的代码  
  
   
  
}  
  
@Cache("SomeObject",key="${id}")  
  
public SomeObject getSomeObject(String id){  
  
    SomeObject someObject=//真实的获取  
  
    return someObject;  
  
}  
```
 

这个时候，程序员们的代码已经非常清爽了，里面不再有与缓冲相关的部分内容，但是引入一个新的问题，就是处理注解的代码怎么写？需要引入容器，比如：Spring，这些Bean必须被容器所托管，如果直接new一个实例，就没有办法用缓冲了。还有一个问题是：程序员的工作量虽然有所节省，但是还是要对程序代码有侵入性，需要引入这些注解，如果要增加超越现有注解的功能，还是需要重新写过这些类，引入其它的注解，修改现有的注解。

所以，上面是个可以接受的方案，但明显还不是很好的方案。

假如有一个程序员火大了，他发出下面的抱怨：“我只管做我的业务，放不放缓冲和我有1毛钱关系么？总因为缓冲的事情让我改来改去，程序改得乱七八糟不说，我的时间，我的工作进度都影响了谁来管？以后和缓冲相关的事情别他妈的来烦我！”，作为架构师的你，你怎么看？最起码，我觉得他是说得非常有道理的。我们再返过头来看看最原始的代码：

```
public void saveSomeObject(SomeObject someObject){  
  
    //下面是真实保存对象的代码  
  
   
  
}  
  
public SomeObject getSomeObject(String id){  
  
    SomeObject someObject=//真实的获取  
  
    return someObject;  
  
}  
```
 

这里是干干净净的业务代码，和缓冲没有一点关系。后来由于性能方面的要求，需要做缓冲，OK，这一点是事实，但是用什么缓冲或怎么缓冲，与程序员确实是没有什么关系的，因此，是不是可以不让程序员参与，就可以优雅的做到添加缓冲功能呢？答案当然是肯定的。 

## 需求整理
代码当中，不要体现与缓冲相关的内容，也就是说做不做缓冲及怎么做缓冲不要影响到业务代码
不管是从容器中取实例还是new实例，都可以同样的起作用，也就是说可以不必依赖具体的容器

## 解决思路：
放不放缓冲、怎么放缓冲、缓冲有效时间等等，这些内容是在运行期发现存在性能瓶颈，然后提交给程序员来进行优化的。为此，我们设计了一个配置来描述这些缓冲相关的声明。

当然，这个配置文件的结构，可以根据自己所采用的缓冲框架来进行相应的定义。

比如：

```
<redis-caches>  
  
  <redis-cache type="org.tinygroup.redis.test.UserDao">  
  
     <redis-method method-name="saveUser">  
  
           <redis-expire value="1000"></redis-expire>  
  
           <redis-string type="user" key="${user.id}" paramter-name="user"><redis-string>  
  
     </redis-method>  
  
  </redis-cache>  
  
  <redis-cache type="org.tinygroup.redis.test.UserDao">  
  
     <redis-method method-name="getUser">  
  
           <redis-expire value="1000"></redis-expire>  
  
           <redis-string type="user" key="${id}" ><redis-string>  
  
     </redis-method>  
  
  </redis-cache>  
  
</redis-caches>  
```
 

我们在实际应用当中，配置比上面的示例更完善，那现在我先讲一下上面的两段配置的含义。

在UserDao的saveUser的时候，会同步的把User数据放到Redis中进行缓冲，缓冲时间为1秒，存放的缓冲数据的类型为user，键值为${user.id}，也就是要保存的用户的主健。实际进入到Redis的时候，在Redis中的健值是由上面类型与key的共同组成的。

在调用UserDao的getUser的时候，会先从缓冲中获取类型为user,键值为${id}的数据，如果缓冲中在，则取出并返回，如果缓冲中没有，则从原有业务代码中取出值并放入缓冲，然后返回此对象。

哇，这个时候，就非常爽了，只要通过声明就可以做到对缓冲的处理了，但是一个问题就出来了，如何实现上面的需求呢？

通过配置文件外置，确实做到了对业务代码的0侵入，但是如何为原有业务增加缓冲相当的业务逻辑呢？由于需求2要求可以new，也可以从容器中获取对象实例，因此利用容器AOP解决的跑就被封死了，因此，就得引入字节码的方式来进行解决。

## 具体实现
写一个基于Maven的缓冲代码处理插件，在编译后增加此处理插件，根据配置文件对原有代码进行扫描并修改其字节码以添加缓冲相关处理逻辑。

现在只要使用Maven进行compile或install就可以自动添加缓冲相关的逻辑到class文件中了。

至此，我们已经分析了缓冲代码直接耦合到代码中，并分析了其中的缺点，最终演化了注解方式，外置配置方式，并简要介绍了实现方法。

具体实现，采用的技术就比较多了，有Maven插件、有Asm、有模板引擎还有Tiny框架的一些基础工程，如:VFS，FileResolver等等。

如果采用Tiny框架，可以直接拿来用，如果不用Tiny框架，可以参照上面的思路做自己的实现。