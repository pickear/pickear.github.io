title: '并发:不一致读带来的问题'
author: Dylan
tags:
  - 并发
  - 不一致读
categories:
  - 软件设计
date: 2018-11-05 15:32:00
---
&emsp;&emsp;关于并发时的不一致读，在前面一篇文章 [<并发的思考>](/2018/10/07/thinking_in_concurrency/#more)有粗略读过。在这里，用一篇文章的篇幅来展开读读这个问题。因为这个问题很普遍，也是很多人会忽略的。首先，用一个时序图开呈现不一致读的产生:

![图片1](/images/blog/concurrency_disagree_1.png)

&emsp;&emsp;首先，A和B先后读取了customer信息并且进行编辑。A在编辑完提交时，B已经完成了编辑和提交。这种情况下，因为B的提交对A是不可见的，所以A最后的提交会覆盖B的提交。出现这个问题的原因，是因为最后提交者不是基于最新的数据进行编辑提交。解决这个问题的最本质方法是，要么后提交者放弃提交，要么重新基于最新数据进行编辑再提交。下面，介绍几种方法避免这个问题的产生。

### 提交时拒绝

&emsp;&emsp;这种方式也被martin fowler乐观离线锁，就是乐观地认为冲突不会发生，到最后真的发生了再处理。首先，为每一份数据定义一个版本号version。
每一次对数据的更新都基于获取数据时的版本号，并且在更新的同时版本号加1.下面是一个更新的SQL:

```shell
UPDATE customer SET name=customer.name,SET version=version+1 WHERE id=customer.id and version=customer.version
```

&emsp;&emsp;这里解释一下这条SQL为什么能避免不一致读问题带来的后果。首先，A,B先后读取了customer的信息，此时，customer的版本号version=1。在B编辑完提交编辑结果时，由于在这之前没有人对数据更新了修改，此时版本号version还是1。当B更新成功后，版本号version=2。此时，由于B已经将version改成了2，A再提交编辑后，更新失败，update的SQL返回0(代表没有更新到数据)。当update语句返回的是0而不是1的时候，程序可以抛一个异常让A的事务回滚，或者做一些其他处理通知A数据不是最新的的，重新加载数据再处理。

&emsp;&emsp;如果觉得每一个表都添加一个version字段不好，可以修改UPDATE语句如下:

```shell
UPDATE customer SET name=customer.name,SET password=customer.password WHERE id=customer.id and name=oldcustomer.name and oldcustomer.password
```
要修改什么属性的值，就在WHERE语句添加该属性条件判断。如果属性之前被修改过，条件判断就不成立，更新就失败。这种方式还是没有添加version属性简单方便。

![图片2](/images/blog/concurrency_disagree_2.png)


&emsp;&emsp;提交时拒绝这种处理方式有个不好的地方，当A在编辑时做了很多的修改，当提交的时候才告诉A提交失败，让A重新加载数据再编辑，对A来说是一件很抓狂的事。当然，在程序上也可以做一些处理，保存A所做的修改，冲突时提示A哪里地方和新的数据有冲突，让A决定如何做。不过，这样也让程序变得更加复杂。并且，如果此类事件频繁发现，对A来说是绝望的。所以这种方式比较适合冲突比较少的情况，如果冲突频繁发生，这种方式就不是很友好了。

### 读取时拒绝

&emsp;&emsp;做了很多事情，到最后被告知都是徒劳，其实是一件让人难以接受的事，特别是还频繁发生。那么我们是否可以，既然注定失败，那就让失败来得更早些？那就是接下来要讲的，读取时拒绝，也就是悲观离线锁。当数据被A读取之后，B就不能再读取。

![图片3](/images/blog/concurrency_disagree_3.png)


&emsp;&emsp;我们可以用锁的方式来实现A读取之后不让任何人再读取。首先，A在读取的时候先要获取一把锁，如果锁获取成功，就允许读取。A持有这把锁直到修改完数据成功提交后再释放。在此期间，B想读取数据，因为无法获取到锁而失败。由于A持有锁可能时间会比较久，所以为了不影响其他人读取其他数据，锁的粒度一定要小，只锁定特定的行数据，不能将整个表数据锁定。这种获取锁的方式可以用分布式锁，也可以用后面介绍的数据库表方式控制锁。

&emsp;&emsp;读取时拒绝的好处是让用户不用丢弃已经完成的工作。那么，读取时拒绝有哪些不好的地方。试想一下，当A获取了数据打开了编辑，此时A由于其他事情走开了，长期没有提交编辑，由于A长时间持有锁不释放，导致对这条数据的编辑长时间不可用，这样就会给用户一种系统长时间崩溃的感觉。所以，读取时拒绝适用于冲突率很高的并发会话，或者冲突处理代价很高的情况下。读取时拒绝只是作为提交时拒绝的一个补充，在真正需要的时候才使用。


### 数据改变时提醒

&emsp;&emsp;提交时拒绝和读取时拒绝都有各自的适用场景，也都有各自好和不好的地方。是否有一种相对更折衷的方式，不会让用户在最后提交时才知道冲突了，或者不会因为别人读取了数据我连读取的权限都没有了，毕竟，我可能只是想看数据，并没有计划编辑数据。那就是DDD里面提到的事件驱动，对数据的增删改，都会抛出一个相应的事件。事件发生后，由事件监听器决定做相应的处理。

![图片4](/images/blog/concurrency_disagree_4.png)


&emsp;&emsp;首先，B对数据修改成功后，抛出一个ModifiedEvent(数据已修改)事件。事件监听器接收到数据已经被修改，就通知A，让A决定是否要加载新数据，或者将修改后的数据与A编辑的数据做一个冲突比较，让A决定使用哪一个版本的数据。在A提交前就已经感知数据的变化，并做出了决定，那么在A提交修改后，就可以直接更新数据，而不需要使用version来让A在提交时失败了。那么如果通知A数据已经改变了，如果是了个WEB应用，A已经将数据加载到页面上，如何通知A呢？可以在页面上做一个定时器，定时到后端检查数据有没有改变。也可以用websocket的方式持有一个长连接，当数据被修改时让后端主动通知。如果有其他的方式，也可以采用。


### 共享的乐观离线锁

&emsp;&emsp;有时候，修改数据并不单单是修改一个表一个实体，而是会修改多个表多个实体，那么，要怎么通过version来避免不一致读的问题？例如要修改用户信息，用户是由Customer和Address组成。是不是Customer和Address都各自有一个verion？还是Address共用Customer的version，用一个粗粒度的锁同时锁住Customer和Address？粗粒度锁覆盖多个对象的单个锁，简化加锁本身，也可以让你不必为给它们加锁而加载所有对象。

&emsp;&emsp;在DDD中提到了聚合的概念。Customer和Address其实是属于同一个聚合，Customer是聚合根，Adress是该聚合里面的值对象。对于聚合外面来说，只能依赖聚合根，要访问聚合根内部的对象，都只能通过聚合根提供接口。这就保证了聚合里面的对象，不会分散到系统各个地方而难以控制。粗粒度锁只要锁住聚合根，就能够锁住整个聚合的对象。


![图片5](/images/blog/concurrency_disagree_5.png)

&emsp;&emsp;首先创建一个version表:

```shell
CREATE TABLE version(id bigint primary key,value bigint);
```

&emsp;&emsp;定义一个Version类:

```java
public class Version{
	private Long id;
	private long value;
	private boolean isNew;
	
	public static Version create(){
		Version version = new Version(IdGenerator.nextId(),0);
		version.isNew = true;
		return version;
	}
	
	public void insert(){
		if(isNew()){
			//这里执行数据库的插入操作，生成version记录。
			//INSERT INTO version(id,value) value(id,value)
			
			isNew = false;
		}
	}
	
	public void increment(){
		//这里更新version记录，记value加1
		//UPDATE SET value=value+1 WHERE id=id and value=value;
		//执行update语句后会返回一个int类型的返回值，代表被更新的记录数，这里例如返回值变量为rowCount
		
		if(rowCount == 0){   //说明版本已经被改变
			throw new RuntimeException("数据已被改变");
		}
		value++;
	}
}



public class Entity{
	private Long id;
	private Version version;
}

public class Customer extends Entity{
	private String name;
	private String password;
	private List<Address> address = new ArrayList();
	
	public static Customer create(String name,String password){
		return new Customer(IdGenerator.nextId(),Version.create(),name,password);
	}
	
	public void addAddress(String city){
		Address address = Address.create(this,city);
		this.address.add(address);
	}
	
}


public class Address extends Entity{
	private Long customerId;
	private String city;
	
	public static Address create(Customer customer,String city){
		return new Address(IdGenerator.nextId(),customer.getVersion(),customer.getId(),city);
	}

}


public class CustomerRepository{

	public void insert(Customer customer){
		customer.getVersion().insert();   //首先在版本号表插入版本记录
		//再插入Customer信息
		//.......
	}
	
	public void update(Customer customer){
		customer.getVersion().increment();  //首先先将版本号加1,如果成功，说明数据没被修改过。
		
		//再更新Customer信息
	}
	
	public void delete(Customer customer){
		//1.先删除所有该Customer的Address数据
		//2.删除Customer数据
		//3.删除version表数据
	}
}
```