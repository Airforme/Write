# MongoDB Reactive Stream使用简介

## 背景知识

### 1.什是响应式编程

**响应式编程就是与异步数据流交互的编程范式.本质上就是[观察者模式](https://www.runoob.com/design-pattern/observer-pattern.html)的一种表现形式**

比如说微信聊天,如果你有没有加对方好友(订阅),对方是不能发送消息给你的;再比如说数据库中的数据,需要一次查找大量的数据,并且我们需要对做进一步的处理,

通常的场景:数据库必须返回所有的数据之后,我们才能开始对数据进行处理;响应式编程:要开始处理数据了,要求数据库给我一条数据,我要开始处理,这一条处理处理完成之后,再要求数据库发给我一条数据,再进行处理 ...... ;这些处理的逻辑执行都是异步的,我们继续做其他的事情

![[逻辑模型|flux.svg]]

如果想进一步了解,自行查找学习

### 2.Reactor

reactor是基于响应式编程的一个框架,4D项目引入的是此框架 [快速入门](https://blog.51cto.com/liukang/2090191), [官方API文档](https://projectreactor.io/docs/core/release/api/)

除了Reactor还有,RxJava, Akka Streams,Vert.x,想详细了解自行查找

## 为什么引入响应式框架

### 1.方便代码编写

响应式编程可以加深你代码抽象的程度，让你可以更专注于定义与事件相互依赖的业务逻辑，而不是把大量精力放在实现细节上，同时，使用响应式编程还能让你的代码变得更加简洁。 比如代码:

```java
public List<ModelProperties> getAllProperties(Integer modelId, MongoDataType type) {
    MongoCollection<Document> collection = ReactiveMongodbUtil.getCollection(modelId,type);
    QueryModelSubscriber<ModelProperties> querySubscriber = new QueryModelSubscriber<>();
    //从MongoDB中查找出所有的数据,并创建一个Flux对象
    Flux.from(collection.find())
    //开启多线程对数据进行并发处理,而我们不需要担心多线程的问题
    .parallel()
    //将每一个Document对象转为ModelProperties对象,n为Document的对象,
    //这一步之后,对象FLux<Document> 将变成 FLux<ModelProperties>
    .map(n->new ModelProperties().parseDocument(n))
    //对每一个ModelProperties对象里面的数据进行处理
    .doOnNext(v->{
        List<ModelPropertyValue> properties = v.getProperties();
        List<ModelPropertyValue> values = new LinkedList<>();
        for (ModelPropertyValue value : properties) {
            if (value.getValue().equals("5")||value.getValue().equals("15")) {
                values.add(value);
            }
        }
        v.setProperties(values);
        System.out.println("set:"+values);
        })
    //被订阅,这一步是必须,如果不被订阅,上述的代码就不会执行
    .subscribe(querySubscriber);
    //阻塞等待数据处理的结果,querySubscriber此订阅者为自己实现的阻塞
    return querySubscriber.get();
}
```

### 2.业务需求

模型的数据量一般会很大,如果将一个模型的数据全部拿到JVM中进行处理,一个模型也许还可以支撑,如果是3个,7个,甚至是10个模型同时处理呢?势必会引起内存泄漏,而导致整个服务奔溃.采用这种响应式的编程,在JVM中我们只存储处理之后的我们需要的数据.

如果处理后的数据量还是很大,我们也可以将订阅出发,交给前端,这个时候我们的服务就会变成数据处理其中的一部分,数据就会直接流到用户端.可以很大程度的提高应用的执行效率,也可以很大程度的提高应用的并发量.当然目前我们的框架技术不支持这种操作,所以,采取了一个折中的办法来处理

## Reactive-MongoDB 的使用

### 1.添加依赖

```pom
<dependencies>
    <dependency>
        <groupId>org.mongodb</groupId>
        <artifactId>mongodb-driver-reactivestreams</artifactId>
        <version>1.12.0</version>
    </dependency>
</dependencies>
```

### 2.连接到MongoDB

```java
// 直接连接到MongoDB服务,默认localhost:27017
MongoClient mongoClient = MongoClients.create();

// Use a Connection String
MongoClient mongoClient = MongoClients.create("mongodb://localhost");

// or a Connection String
MongoClient mongoClient = MongoClients.create(new ConnectionString("mongodb://localhost"));

// or provide custom MongoClientSettings
ClusterSettings clusterSettings = ClusterSettings.builder().hosts(asList(new ServerAddress("localhost"))).build();
MongoClientSettings settings = MongoClientSettings.builder().clusterSettings(clusterSettings).build();
MongoClient mongoClient = MongoClients.create(settings);

MongoDatabase database = mongoClient.getDatabase("mydb");
```

对于getDatabase(“mydb”)，不需要连接到MongoDB。但是MongoDatabase实例提供了与数据库交互的方法，但数据库可能并不实际存在，只会在通过某种方式插入数据时创建;例如，建立一个集合或插入文件。 MongoClient实例实际上表示给定MongoDB服务器部署的连接池;即使有多个并发执行的异步操作，您也只需要一个类MongoClient实例。

通常，您只需为给定的数据库集群创建一个MongoClient实例，并在应用程序中使用它。创建多个实例时:

所有资源使用限制(最大连接等)适用于每个MongoClient实例要释放一个实例，请确保调用了MongoClient.close()来清理资源

### 3.获取集合

要获取要操作的集合，请将集合的名称指定给getCollection(String collectionName)方法:

```java
//获取一个test集合
MongoCollection<Document> collection = database.getCollection("test");
```

### 4.插入数据

```json
{
   "name" : "MongoDB",
   "type" : "database",
   "count" : 1,
   "info" : {
               x : 203,
               y : 102
             }
}
```

将上述的一个document插入数据库中的代码如下:

```java
Document doc = new Document("name", "MongoDB")
               .append("type", "database")
               .append("count", 1)
               .append("info", new Document("x", 203).append("y", 102));
collection.insertOne(doc).subscribe(new OperationSubscriber<Success>());
```

在API中，所有返回发布者的方法都是“冷”流，这意味着在它们被订阅之前什么都不会发生。如下代码,就不会发生任何事

```java
Publisher<Success> publisher = collection.insertOne(doc);
```

只有当publisher 被订阅去请求数据才将发生操作:

```java
Publisher<Success> publisher = collection.insertOne(doc);
publisher.subscribe(new Subscriber<Success>() {
    @Override
    public void onSubscribe(final Subscription s) {
        s.request(1);  // <--- Data requested and the insertion will now occur
    }

    @Override
    public void onNext(final Success success) {
        System.out.println("Inserted");
    }

    @Override
    public void onError(final Throwable t) {
        System.out.println("Failed");
    }

    @Override
    public void onComplete() {
        System.out.println("Completed");
    }
});
```

一旦文档被插入，onNext方法将被调用，它将打印“Inserted”，然后是onComplete方法，它将打印“Completed”。如果由于任何原因出现错误，onError方法将打印“Failed”。

#### 插入多个文档

```java
List<Document> documents = new ArrayList<Document>();
for (int i = 0; i < 100; i++) {
    documents.add(new Document("i", i));
}

subscriber = new ObservableSubscriber<Success>();
collection.insertMany(documents).subscribe(subscriber);
subscriber.await();//阻塞,等待插入操作完成
```

### 5.查询

#### 要获取集合中的第一个文档

请调用find()操作上的第一个()方法。

```java
subscriber = new PrintDocumentSubscriber();
collection.find().first().subscribe(subscriber);
subscriber.await();
```

输出:

```json
{ "_id" : { "$oid" : "551582c558c7b4fbacf16735" },
  "name" : "MongoDB",
  "type" : "database",
  "count" : 1,
  "info" : { "x" : 203, "y" : 102 } }
```

\_id元素已由MongoDB自动添加到您的文档中。MongoDB保留以“_”和“$”开头的字段名供内部使用。

#### 获取集合中所有的文档

```java
subscriber = new PrintDocumentSubscriber();
collection.find().subscribe(subscriber);
subscriber.await();
```

获取带有查询筛选器的文档

```java
import static com.mongodb.client.model.Filters.*;
//  i==71
collection.find(eq("i", 71)).first().subscribe(new PrintDocumentSubscriber());
//  50<i<100
collection.find(and(gt("i", 50), lte("i", 100))).subscribe(new PrintDocumentSubscriber());
```

将会输出:

```json
{ "_id" : { "$oid" : "5515836e58c7b4fbc756320b" }, "i" : 71 }
```

使用 Filters, Sorts 和 Projections 帮助程序以简单而简洁的方式构建查询。 对文档数据排序 //存在字段i的文档数据进行排序

```java
collection.find(exists("i")).sort(descending("i")).subscribe(new PrintDocumentSubscriber());
```

有时我们不需要文档中包含的所有数据。Projections可用于为查找操作构建投影参数并限制返回的字段。下面我们将查找排除_id字段并输出第一个匹配的文档:

```java
collection.find().projection(excludeId()).subscribe(new PrintDocumentSubscriber());
```

### 6.更新数据

MongoDB支持许多[更新操作符](https://docs.mongodb.com/manual/reference/operator/update-field/)。 只更新一条数据:我们更新第一个文档，满足过滤器i等于10，并设置i的值为110:

```java
collection.updateOne(eq("i", 10), new Document("$set", new Document("i", 110)))
          .subscribe(new PrintSubscriber<UpdateResult>("Update Result: %s"));
```

要更新与筛选器匹配的所有文档，请使用updateMany方法。这里我们将当i小于100时i的值加100

```java
collection.updateMany(lt("i", 100), new Document("$inc", new Document("i", 100)))
          .subscribe(new PrintSubscriber<UpdateResult>("Update Result: %s"));
```

### 7.删除数据

要删除最多一个文档使用deleteOne方法

```java
collection.deleteOne(eq("i", 110))
          .subscribe(new PrintSubscriber<DeleteResult>("Delete Result: %s"));
```

要删除与筛选器匹配的所有文档，请使用deleteMany方法。这里我们删除所有i大于或等于100的文档:

```java
collection.deleteMany(gte("i", 100)
          .subscribe(new PrintSubscriber<DeleteResult>("Delete Result: %s"));
```

### 8.获取数据库列表

使用PrintSubscriber来打印数据库名称列表

```java
mongoClient.listDatabaseNames().subscribe(new PrintSubscriber<String>("Database Names: %s"));
```

### 9.删除一个数据库

```java
subscriber = new ObservableSubscriber<Success>();
mongoClient.getDatabase("databaseToBeDropped").drop().subscribe(subscriber);
subscriber.await();
```

### 11.创建一个集合

MongoDB中的集合是通过简单地将文档插入其中来自动创建的。使用createCollection方法，您还可以显式地创建一个集合，以便自定义其配置。例如，创建一个大小为1m的上限集合:

```java
database.createCollection("cappedCollection", new CreateCollectionOptions().capped(true).sizeInBytes(0x100000))
    .subscribe(new PrintSubscriber<Success>("Creation Created!"));
```

### 12.获取集合列表

得到一个数据库中可用集合的列表:

```java
database.listCollectionNames().subscribe(new PrintSubscriber<String>("Collection Names: %s"));
```

### 13.删除一个集合

```java
subscriber = new ObservableSubscriber<Success>();
collection.drop().subscribe(subscriber);
subscriber.await();
```

### 14.创建一个索引

MongoDB支持二级索引。要创建索引，只需指定字段或字段的组合，并为每个字段指定该字段的索引方向。升序为1，降序为-1。下面在i字段上创建一个升序索引:

```java
// create an ascending index on the "i" field
collection.createIndex(new Document("i", 1)).subscribe(new PrintSubscriber<String>("Created an index named: %s"));
```

### 15.获取集合的索引列表

使用listIndexes()方法获得索引列表:

```java
collection.listIndexes().subscribe(new PrintDocumentSubscriber());
```

## Reactor 使用

### [Flux](https://projectreactor.io/docs/core/release/api/) 和 [Mono](https://projectreactor.io/docs/core/release/api/)

Flux 和 Mono 是 Reactor 中的两个基本概念。

Flux 表示的是**包含 0 到 N 个元素的异步序列**。在该序列中可以包含三种不同类型的消息通知：正常的包含元素的消息、序列结束的消息和序列出错的消息。当消息通知产生时，订阅者中对应的方法 onNext(), onComplete()和 onError()会被调用。

Mono 表示的是**包含 0 或者 1 个元素的异步序列**。该序列中同样可以包含与 Flux 相同的三种类型的消息通知。Flux 和 Mono 之间可以进行转换。对一个 Flux 序列进行计数操作，得到的结果是一个 Mono< Long >对象。把两个 Mono 序列合并在一起，得到的是一个 Flux 对象。

### 创建 Flux

**just** 可以指定序列中包含的全部元素。创建出来的 Flux 序列在发布这些元素之后会自动结束。

**fromArray，fromIterable和 fromStream** 可以从一个数组、Iterable 对象或 Stream 对象中创建 Flux 对象。

**empty** 创建一个不包含任何元素，只发布结束消息的序列。

**error(Throwable error)** 创建一个只包含错误消息的序列。

**never** 创建一个不包含任何消息通知的序列。

**range(int start, int count)** 创建包含从 start 起始的 count 个数量的 Integer 对象的序列。

**interval(Duration period)和 interval(Duration delay, Duration period)** 创建一个包含了从 0 开始递增的 Long 对象的序列。其中包含的元素按照指定的间隔来发布。除了间隔时间之外，还可以指定起始元素发布之前的延迟时间。

**intervalMillis(long period)和 intervalMillis(long delay, long period)** 与 interval()方法的作用相同，只不过该方法通过毫秒数来指定时间间隔和延迟时间。

示例:

```java
Flux.just("Hello", "World").subscribe(System.out::println);
Flux.fromArray(new Integer[] {1, 2, 3}).subscribe(System.out::println);
Flux.empty().subscribe(System.out::println);
Flux.range(1, 10).subscribe(System.out::println);
Flux.interval(Duration.of(10, ChronoUnit.SECONDS)).subscribe(System.out::println);
Flux.intervalMillis(1000).subscribe(System.out::println);
```

### 操作符

#### 转换操作符

常用的转化操作符包括 buffer,map,flatMap和window等

**buffer** 把当前流中的元素收集到集合中，并把集合对象作为流中的新元素。

**map** 相当于一种映射操作,对流中的每个元素应用一个映射函数,从而达到变换的效果.建议使用flatMap

**flatMap** 把流中的每一个元素转换成一个流,再把转换之后得到的所有流中的元素合并

**window** 类似于buffer,不同的是window操作符是把当前流中的元素收集到另外的Flux序列中,返回值是Flux<Flux< T >>

#### 过滤操作符

常用过滤操作符有: filter,first,last,skip/skipLast,take/takeLast 等

**filter** 过滤器,只留下符合条件的元素

**first** 返回六中的第一个元素

**last** 返回流中的最后一个元素

**skip/skipLast** skip会忽略数据流前面的n个元素,skipLast会忽略数据流后面的n个元素

**take/takeLast** 从当前流中提取元素,takeLast从当前流中的尾部提取元素

#### 组合操作符

常用组合操作符 then/when,merge,startWith和zip等

**then/when** then等上一个操作完成再做下一个,when等多个操作一起完成

**merger** 把多个流合并成一个Flux序列

**startWith** 在数据元素序列的开头插入指定的元素项

**zip** 把当前流中的元素与另外一个流中的元素按照一对一的方式进行合并

####  

#### 条件操作符

常用条件操作符 defaultEmpty,skipUtil,skipWhile,takeUtil和takeWhile等

**defaultEmpty** 返回原始数据流的元素,如果没有元素,则返回一个默认的元素

**skipUtil** skipUtil(Predicate<? super T> predicate)开始一直丢弃原始数据流中的元素,直到Predicate为true

**takeUtil** takeUtil(Predicate<? super T> predicate)开始一直提取原始数据流中的元素,直到Predicate为true

**takeWhile** takeWhile(Predicate<? super T> predicate)当Predicate为true时,才提取此元素

**skipWhile** skipWhile(Predicate<? super T> predicate)当Predicate为true时,才丢弃此元素

#### 数学操作符

常用数学操作符: concat,count 和 reduce

**concat** 采用顺序的方式合并来自不同的Flux数据

**count** 用来统计Flux中的元素的个数

**reduce** 对流中的所有元素进行累积操作,得到一个Mono序列

#### Observable工具操作符

常用Observable工具操作符: delay,subscribe,timeout等

**delay** 将事件的传递延迟一段时间

**subscribe** 添加相应的订阅逻辑

**timeout** 在特定的时间段内没有产生任何时间时,将生成一个异常

**block** 在接受下一个元素之前一直阻塞

详细API讲解请查看: [Flux](https://projectreactor.io/docs/core/release/api/) 和 [Mono](https://projectreactor.io/docs/core/release/api/)

## 项目中代码示例

请查看类: **com.util.mongoDB.MongoOpsOfModel**

### 代码简单解释

```java
public List<ModelProperties> getAllProperties(Integer modelId, MongoDataType type) {
    //获取MongoDB的连接,已实现分库分表的操作,type为模型的那类数据,目前有默认,工程量,5D模拟
    MongoCollection<Document> collection = ReactiveMongodbUtil.getCollection(modelId,type);
    //自定义的订阅者
    QueryModelSubscriber<ModelProperties> querySubscriber = new QueryModelSubscriber<>();
    //从MongoDB中查找出所有的数据,并创建一个Flux对象
    Flux.from(collection.find()
             //过滤条件(properties.value == 2||properties.value==5||properties.value>17
            .filter(Filters.or(
                    Filters.eq(ModelProperties.Properties+"."+ModelPropertyValue.Value, 2),
                    Filters.gte(ModelProperties.Properties+"."+ModelPropertyValue.Value, 17),
                    Filters.eq(ModelProperties.Entity_Id, 5)))
            //只返回字段 entity_id,properties.name,properties.value,properties.description
            .projection(Projections.include(ModelProperties.Entity_Id,
                    ModelProperties.Properties+"."+ModelPropertyValue.Name,
                    ModelProperties.Properties+"."+ModelPropertyValue.Value,
                    ModelProperties.Properties+"."+ModelPropertyValue.Description))
            //按照entity_id正序排序
            .sort(Sorts.ascending(ModelProperties.Entity_Id))
            //为本次查询添加注释
            .comment("带过滤器和指定字段的查询"))
    //开启多线程对数据进行并发处理,而我们不需要担心多线程的问题
    .parallel()
    //将每一个Document对象转为ModelProperties对象,n为Document的对象,
    //这一步之后,对象FLux<Document> 将变成 FLux<ModelProperties>
    .map(n->new ModelProperties().parseDocument(n))
    //对每一个ModelProperties对象里面的数据进行处理
    .doOnNext(v->{//lambda表达式写法,只针对一个interface中只包含一个需要实现的方法的情况
        //对ModelProperties对象中的properties里面的数据进行进一步处理
        List<ModelPropertyValue> properties = v.getProperties();
        List<ModelPropertyValue> values = new LinkedList<>();
        for (ModelPropertyValue value : properties) {
            //ModelPropertyValue对象的value值为5或者15
            if (value.getValue().equals("5")||value.getValue().equals("15")) {
                values.add(value);
            }
        }
        v.setProperties(values);
        System.out.println("set:"+values);
        })
    //被订阅,这一步是必须,如果不被订阅,上述的代码就不会执行
    .subscribe(querySubscriber);
    //阻塞等待数据处理的结果,querySubscriber此订阅者为自己实现的阻塞
    return querySubscriber.get();
}

```

自定义的订阅者

```java
public class QueryModelSubscriber<T> implements Subscriber<T>{

	//响应数据
    private final List<T> received;
    //错误信息
    private final List<Throwable> errors;
    //等待对象
    private final CountDownLatch latch;
    //订阅器
    private volatile Subscription subscription;
    //是否完成
    private volatile boolean completed;
    
    public QueryModelSubscriber() {
        this.received = new ArrayList<T>();
        this.errors = new ArrayList<Throwable>();
        this.latch = new CountDownLatch(1);
    }
    
	@Override
	public void onSubscribe(Subscription s) {
		this.subscription = s;
		
	}
	@Override
	public void onNext(T t) {
//		System.out.println(Thread.currentThread().getName()+"---"+t);
		received.add(t);
	}
	@Override
	public void onError(Throwable t) {
		System.err.println(t);
		errors.add(t);
		onComplete();
	}
	@Override
	public void onComplete() {
		completed = true;
		latch.countDown();
	}

	public List<T> get(final long timeout,final TimeUnit unit){
		return await(timeout,unit).getReceived();
	}
	
	public List<T> get(){
		return await().getReceived();
	}
	
	/**
     * 	一直阻塞等待请求完成
     *
     * @return
     * @throws Throwable
     */
    public QueryModelSubscriber<T> await(){
        return await(Long.MAX_VALUE, TimeUnit.MILLISECONDS);
    }

	/**
	 * 阻塞一定时间等待完成
	 * 
	 * @param timeout
	 * @param unit
	 * @return
	 * @throws Throwable
	 */
	private QueryModelSubscriber<T> await(long timeout, TimeUnit unit) {
		this.subscription.request(Integer.MAX_VALUE);
		try {
			if (!this.latch.await(timeout, unit)) {
			    throw new MongoTimeoutException("Publisher onComplete timed out");
			}
			if (!this.errors.isEmpty()) {
			    throw errors.get(0);
			}
		} catch (Throwable e) {
			e.printStackTrace();
			throw new RuntimeException("查询失败 :" +e.getMessage());
		}
        return this;
	}

	
	private List<T> getReceived() {
		return received;
	}

	public List<Throwable> getErrors() {
		return errors;
	}

	public CountDownLatch getLatch() {
		return latch;
	}

	public Subscription getSubscription() {
		return subscription;
	}

	public boolean isCompleted() {
		return completed;
	}

}
```

参考示例代码,根据自己的业务编写自己需要的MongoDB的操作代码,其中自定义的订阅者可复用

响应式编程是一种新的技术,JDK9中已经添加响应式编程到JDK源码中,如有什么疑问,一起共同探讨学习
