# 业务流程引擎设计

一般的时候，我们都采用编程式开发，编程式开发的好处非常明显：直接、高效、自由，当然其缺点也是有的，与其优点刚好相对，因为直接，所以有些变化都要进行代码上的修改；因为高效，所以一旦出问题，导致的结果也比较严重，因为自由，所以带来的修改风险也比较大。  这也就是许多大的公司都在进行流程化开发的重要原因之一，比如：上海普元，Livebos, Justep，还有许许多多知名不知名的公司都有类似的流程化开发引擎存在，通过流程化开发，增强代码的复用性，降低软件开发成本及测试成本，提升软件的可维护性及降低维护成本。 

在设计Tiny框架时，我们也考虑了自己的方案，主要包括以下几个方面的问题： 

### a.组件扩充的便捷性 
组件的扩充的便捷性是指，流程其实玩的就是组件，如果组件扩充起来非常困难，会直接影响到流程引擎的可用性。所以Tiny框架的流程引擎的组件结构非常之简单，仅有一个接口方法；流程组件的注册与加载也是非常重要的，如果在扩充流程组件的时候，需要复杂的注册或配置过程，这个时候流程扩充的便捷性也会大大降低。Tiny框架采用了引用即注册的方案，只要把流程组件放入系统运行环境之间，就完成了流程组件的注册，即可以在流程中使用，便得流程组件的扩充的便捷性大大提高。 

### b.流程的面向对象特性支持  
流程的面向特性支持是指在Tiny框架中流程是具有面向对象的特性的。流程可以进行继承，这样带来一个好处就是多个流程中重复的部分，可以定义在一个父流程中，然后子流程只要继承父流程，即可；流程节点是可以被覆盖的，也就是说，在父流程中可以定义一个空节点，但是流程中定义了流转关系，但是流程节点的实现留在子流程中实现； 

### c.流程的易编辑性 
流程的编辑必须方便、容易，有专门的流程编辑工具更好，没有的时候，使用普通的Xml编辑器也可以方便的进行编辑。 

### d.流程的可重入性 
一般的流程引擎都是不可重入的，也就是只能从开始执行，执行到结束结点之后完成。Tiny流程引擎支持流程重入，也就是说，不一定是从开始结点执行，可以从任意一个结点执行。这个机制为程序的逻辑提供了非常大的自由度，可以利用此特性容易的构建页面流引擎或工作流引擎。即使是业务流程引擎，也会由此获得更大的自由度。 

由于支持流程的可重入性，在本流程处理当中，不仅可以在当前流程中进行切换与转接，还可以流转到其他流程的节点当中，这在业务处理及页面处理，流程处理方面都提供了极大的使得，但是这也是一个双刃剑，在提供了这么灵活的功能的同时，也会导致业务流程看起来比较复杂，因此，控制方面最好由架构师或核心开发人员来编写，普通开发人员只开发具体的业务点即可。

呵呵，说了这么多，大家理解起来可能还是比较抽象，那就来个例子看看：

 

```
<flow id="1000" name="Hello">  
     <nodes>  
           <node id="begin">  
                 <component class-name="org.tinygroup.flow.HelloWorldComponent">  
                     <properties>  
                         <property name="name" value="world" />  
                     </properties>  
                 </component>  
           </node>  
     </nodes>  
< /flow>  
```

HelloWorldComponent的源码如下：

```
public class HelloWorldComponent implements ComponentInterface {  
     String name;  
     public String getName() {  
         return name;  
     }  
     public void setName(String name) {  
         this.name = name;  
     }  
     public void execute(Context context) {  
         context.put("result", String.format("Hello, %s", name));  
     }  
 }  
```

可以看出，所有组件必须实现ComponentInterface 接口 
从其实现逻辑可以看出，它就是把“Hello, ”加上输入的名字，放在了环境变量的result当中。 下面看看执行结果： 
a.按默认开始结点开始执行

```
Context context = new ContextImpl();  
 flowExecutor.execute("1000",  context);  
 assertEquals("Hello, world", context.get("result"));  
```

b.从指定节点开始执行 

```
Context context = new ContextImpl();  
 flowExecutor.execute("1000","begin", context);  
 assertEquals("Hello, world", context.get("result"));  
```

可以看到确实是执行并返回了结果，但是它的执行机理是怎么样的呢？？ 
实际上，上面的流程是一个简化的流程，就是说Tiny流程引擎的有些参数不输入，也可以按照约定正确的执行，实际上写得完整的话，例子是下面这个样子的：  

```
<flow id="1000" version="1.0" privateContext="false" extend-flow-id="" name="Hello" title="你好示例" default-node-id="end" begin-node-id="begin" end-node-id="end" enable="true">  
   <description>some thing....</description>  
   <nodes>  
     <node id="begin">  
       <component class-name="org.tinygroup.flow.HelloWorldComponent">  
         <properties>  
           <property name="name" value="world"/> <span></span> </properties>  
       </component>  
       <next-nodes>  
         <next-node exception-type="java.lang.Exception" next-node-id="end"/>  
       </next-nodes>  
     </node>  
   </nodes>  
< /flow>  
```

其中flow节点的属性含义为：

- id，唯一确定一个流程
- privateContext，如果是true，则在流程单独申请一个context，否则共用调用者的context，这样可以有效避免环境变量冲突问题
- extend-flow-id，继承的流程id，这个继承id是一个非常强大的功能，后面详细介绍
- version版本号，同一id的流程可以存在多个版本，访问时，如果不指定版本则默认采用最新版本
- name,title仅用于说明其英文，中文名称，易于理解而已。
- default-node-id表示，默认执行节点，即如果一个组件执行完毕，其项值没有指定下一处理节点则执行默认节点
- begin-node-id，开始节点
- end-node-id，结束节点

如果不指定，则begin-node-id默认为begin,end-node-id默认为end

- node节点：id必须指定，在一个流程当中id必须唯一。
- component节点
- class-name用于指定组织实现类名
- properties是组件的属性列表
- property中的name与value是组件的属性的值，value，这里传入的是个字符串，但是实际当中可以处理中可以非常灵活，后面再介绍。
- next-nodes，是指根据执行结果进行后续处理的规则。 
- next-node，具体的一条规则，component-result，匹配项，支持正则表达式，节点中的组件执行结果进行匹配，匹配成功则执行此规则中的下一节点。
- exception-type是异常的类名称，如果出现异常且与这里定义的类型匹配，则执行此规则中的下一节点。 

上面说到继承，流程继承实现起来是非常简单的，只要在extend-flow-id属性中指定即可。

继承不支持多继承，即流程只能继承自一个流程，但是可以支持多层继承，即

a>b>c>d.....

实际开发过程中，不要把继承搞得太复杂，这样会把程序逻辑搞得更难理解的。

继承实际会起到什么作用呢？

首先，会继承一些属性，另外会把节点信息继承过来。
简单来说就是：两者都有，当前流程说了算，当前没有，父流程说了算。

继承应用到什么场景呢？？

继承应用于业务处理的模式非常相似，只有中间处理环境不同的时候。
比如：

A  B  C  D ---O--- -D -C -B -A

类型的业务处理流程，只有O不同，其他处理模式完全相同，此时采用继承方式都非常舒服了，只要定义父流程，在子流程中只用定义O一个流程节点即可。以后要统一进行流程调整，只要在父流程中进行调整就可以了。

比如：flow aa定义为 

```
<flow id="aa" name="aa">  
   <nodes>  
     <node id="begin">  
       <next-nodes>  
         <next-node component-result="begin" next-node-id="hello"/>  
       </next-nodes>  
     </node>  
     <node id="hello">  
       <component class-name="org.tinygroup.flow.HelloWorldComponent">  
         <properties>  
           <property name="name" value="world"/>  
         </properties>  
       </component>  
       <next-nodes>  
         <next-node next-node-id="end"/>  
       </next-nodes>  
     </node>  
   </nodes>  
< /flow>  
```

 flow bb定义为 

```
<flow id="bb" name="bb" extend-flow-id="aa">  
< nodes>  
< node id="hello">  
< component class-name="org.tinygroup.flow.HelloWorldComponent">  
< properties>  
< property name="name" value="world" />  
< /properties>  
< /component>  
< /node>  
< /nodes>  
< /flow>  
```

则流程bb也可以顺利执行，且执行结果是Hello, world  
非常重要的一个亮点就是属性赋值。  
属性赋值是否好用，决定了框架的易用性。  
可以支持常量赋值"1"表示数字常量  
aa 表示字符串常量可以支持，环境变量赋值  
比如：xx表示从环境变量取xx键值的对象  
可以支持属性赋值  
比如：xx.abc表示取环境变量xx的属性abc  
比如：xx.abc.def表示取环境变量xx的属性abc的属性def  
可以支持组合赋值  
比如：${in:aa.abc.def}-${in:bb.cc.dd}    
表示把环境aa中的属性abc的属性def中间加"-"再加上环境变量bb中的cc的属性的dd属性  
其中属性的层次不受限制。  
另外，取值方式，也支持自行扩展：  
比如：可以用${in:xmlkey.aa}也取在环境中xmlkey对应的xml节点的aa属性  
所以，只有想不到的，没有做不到的。   
应用开发与部署方式，比较典型的有B/S与B/A/S，C/A/S等。对于B/A/S和C/A/S方式，因为A与B和C是分离部署的，所以，所有的内容都需要是通过Context进行传递的。  
如果是通过分离式部署，那么就需要通过网络来传递请求环境数据。  
如果是想通过B/S环境来构建系统，此时就会期望通过HTTP处理线程来同布调用流程处理结果。  
同时，有时流程处理的数据可能是在Request,RequestAttribute,Session，Cookie中，如果把这些数据COPY到环境当中去，其实是有较大的性能消耗的。  

本流程引擎即支持通过服务方式调用，也可以通过短路方式进行调用。  
虽然我们推荐使用B/A/S体系架构，但是不能否认，目前我们的许多产品还是在B/S架构下运行的。  

但是好在，这个对于流程引擎来说，他并不直接访问Request和Session,Cookie等内容，所以，即使是集成在一起部署，也不妨碍进行分离式部署，依然可以保证服务的无状态特性，前提就是需要实现一个Context的接口。  

## 小结： 
Tiny的流程引擎，提供了相当强悍的功能及扩展性，上面只说了一部分，有些也没有完全说清楚，实际上，还提供了包含EL表达式等许多高级功能，对于期望进行流程式编排开发来说，有相当好的支持。  从后期效果来看，在Tiny框架中，业务流程编排及页面流程编排都是基于此引擎构建，应用效果非常良好。