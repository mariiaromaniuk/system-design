
<!-- MarkdownTOC -->

- [Using oversell problem as an example](#using-oversell-problem-as-an-example)
	- [Case 1: Subtract purchased count in program and update database to target count](#case-1-subtract-purchased-count-in-program-and-update-database-to-target-count)
	- [Case 2: Decrement database count with purchased count](#case-2-decrement-database-count-with-purchased-count)
- [Standalone lock](#standalone-lock)
	- [Synchronized lock](#synchronized-lock)
		- [Method](#method)
		- [Block](#block)
	- [Reentrant lock](#reentrant-lock)
- [Distributed lock](#distributed-lock)
	- [Use cases](#use-cases)
	- [Requirementss](#requirementss)
	- [CP model](#cp-model)
		- [Comparison](#comparison)
		- [Database](#database)
			- [Ideas behind the scene](#ideas-behind-the-scene)
			- [Approach](#approach)
			- [Pros and Cons](#pros-and-cons)
		- [Zookeeper](#zookeeper)
		- [etcd](#etcd)
			- [Operations](#operations)
	- [AP model - Redis](#ap-model---redis)
		- [Internals](#internals)
		- [Efficiency implementation](#efficiency-implementation)
			- [Acquire lock](#acquire-lock)
			- [Release lock](#release-lock)
			- [Pros](#pros)
			- [Cons](#cons)
				- [Limited use cases](#limited-use-cases)
				- [Link to history on RedLock](#link-to-history-on-redlock)

<!-- /MarkdownTOC -->

# Using oversell problem as an example

## Case 1: Subtract purchased count in program and update database to target count

```
┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ 
                        Program                        │
│                                                       
     ┌──────────────────────────────────────────┐      │
│    │    Get the number of inventory items     │       
     └──────────────────────────────────────────┘      │
│                          │                            
                           ▼                           │
│     ┌─────────────────────────────────────────┐       
      │Subtract the number of purchased items to│      │
│     │           get target count B            │       
      │                                         │      │
│     └─────────────────────────────────────────┘       
 ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┬ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
                           │                            
                           │                            
┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ 
                           ▼                           │
│     ┌─────────────────────────────────────────┐       
      │    Update database to target count B    │      │
│     └─────────────────────────────────────────┘       
                                                       │
│                       Database                        
 ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
```

## Case 2: Decrement database count with purchased count

```
┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ 
                        Program                        │
│                                                       
     ┌──────────────────────────────────────────┐      │
│    │    Get the number of inventory items     │       
     └──────────────────────────────────────────┘      │
│                          │                            
                           │                           │
└ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ 
                           │                            
                           ▼                            
┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐ 
                                                        
│     ┌─────────────────────────────────────────┐     │ 
      │     Decrement database count with B     │       
│     └─────────────────────────────────────────┘     │ 
                                                        
│                      Database                       │ 
 ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  
```

# Standalone lock
## Synchronized lock
### Method

```
	@Transactional(rollbackFor = Exception.class)
    public synchronized Integer createOrder() throws Exception
    {
        Product product = null;

        // !!! Manual transaction management is required. Otherwise, 
        TransactionStatus transaction1 = platformTransactionManager.getTransaction(transactionDefinition);
        product = productMapper.selectByPrimaryKey(purchaseProductId);
        if (product==null)
        {
            platformTransactionManager.rollback(transaction1);
            throw new Exception("item："+purchaseProductId+"does not exist");
        }

        // current inventory
        Integer currentCount = product.getCount();
        System.out.println(Thread.currentThread().getName()+"number of inventory："+currentCount);
 
        // check against inventory
        if (purchaseProductNum > currentCount)
        {
            platformTransactionManager.rollback(transaction1);
            throw new Exception("item"+purchaseProductId+"only has"+currentCount+" inventory，not enough for purchase");
        }

        productMapper.updateProductCount(purchaseProductNum,"xxx",new Date(),product.getId());
        platformTransactionManager.commit(transaction1);

        TransactionStatus transaction = platformTransactionManager.getTransaction(transactionDefinition);
        Order order = new Order();
        order.setOrderAmount(product.getPrice().multiply(new BigDecimal(purchaseProductNum)));
        order.setOrderStatus(1);//待处理
        order.setReceiverName("xxx");
        order.setReceiverMobile("13311112222");
        order.setCreateTime(new Date());
        order.setCreateUser("xxx");
        order.setUpdateTime(new Date());
        order.setUpdateUser("xxx");
        orderMapper.insertSelective(order);

        OrderItem orderItem = new OrderItem();
        orderItem.setOrderId(order.getId());
        orderItem.setProductId(product.getId());
        orderItem.setPurchasePrice(product.getPrice());
        orderItem.setPurchaseNum(purchaseProductNum);
        orderItem.setCreateUser("xxx");
        orderItem.setCreateTime(new Date());
        orderItem.setUpdateTime(new Date());
        orderItem.setUpdateUser("xxx");
        orderItemMapper.insertSelective(orderItem);
        platformTransactionManager.commit(transaction);
        return order.getId();
    }
```

### Block

```
	// OrderService.java

	private Object object = new Object();
	
	@Transactional(rollbackFor = Exception.class)
    public Integer createOrder() throws Exception{
        Product product = null;
		synchronized(OrderService.class) // synchronized(this)
		{
			TransactionStatus transaction1 = platformTransactionManager.getTransaction(transactionDefinition);
	        product = productMapper.selectByPrimaryKey(purchaseProductId);
	        if (product==null)
	        {
	            platformTransactionManager.rollback(transaction1);
	            throw new Exception("item："+purchaseProductId+"does not exist");
	        }

	        // current inventory
	        Integer currentCount = product.getCount();
	        System.out.println(Thread.currentThread().getName()+"number of inventory："+currentCount);
	 
	        // check against inventory
	        if (purchaseProductNum > currentCount)
	        {
	            platformTransactionManager.rollback(transaction1);
	            throw new Exception("item"+purchaseProductId+"only has"+currentCount+" inventory，not enough for purchase");
	        }

	        productMapper.updateProductCount(purchaseProductNum,"xxx",new Date(),product.getId());
	        platformTransactionManager.commit(transaction1);

	        TransactionStatus transaction = platformTransactionManager.getTransaction(transactionDefinition);
	        Order order = new Order();
	        order.setOrderAmount(product.getPrice().multiply(new BigDecimal(purchaseProductNum)));
	        order.setOrderStatus(1); // Wait to be processed
	        order.setReceiverName("xxx");
	        order.setReceiverMobile("13311112222");
	        order.setCreateTime(new Date());
	        order.setCreateUser("xxx");
	        order.setUpdateTime(new Date());
	        order.setUpdateUser("xxx");
	        orderMapper.insertSelective(order);

	        OrderItem orderItem = new OrderItem();
	        orderItem.setOrderId(order.getId());
	        orderItem.setProductId(product.getId());
	        orderItem.setPurchasePrice(product.getPrice());
	        orderItem.setPurchaseNum(purchaseProductNum);
	        orderItem.setCreateUser("xxx");
	        orderItem.setCreateTime(new Date());
	        orderItem.setUpdateTime(new Date());
	        orderItem.setUpdateUser("xxx");
	        orderItemMapper.insertSelective(orderItem);
	        platformTransactionManager.commit(transaction);
	        return order.getId();
		}        
    }
```

## Reentrant lock

```
	private Lock lock = new ReentrantLock();

	@Transactional(rollbackFor = Exception.class)
    public Integer createOrder() throws Exception{
        Product product = null;

        lock.lock();
        try 
        {
            TransactionStatus transaction1 = platformTransactionManager.getTransaction(transactionDefinition);
            product = productMapper.selectByPrimaryKey(purchaseProductId);
            if (product==null)
            {
                platformTransactionManager.rollback(transaction1);
                throw new Exception("item："+purchaseProductId+"does not exist");
            }

            // current inventory
            Integer currentCount = product.getCount();
            System.out.println(Thread.currentThread().getName()+"number of inventory："+currentCount);
 
            // check against inventory
            if (purchaseProductNum > currentCount)
            {
                platformTransactionManager.rollback(transaction1);
                throw new Exception("item"+purchaseProductId+"only has"+currentCount+" inventory，not enough for purchase");
            }

            productMapper.updateProductCount(purchaseProductNum,"xxx",new Date(),product.getId());
            platformTransactionManager.commit(transaction1);
        }
        finally 
        {
            lock.unlock();
        }

        TransactionStatus transaction = platformTransactionManager.getTransaction(transactionDefinition);
        Order order = new Order();
        order.setOrderAmount(product.getPrice().multiply(new BigDecimal(purchaseProductNum)));
        order.setOrderStatus(1);//待处理
        order.setReceiverName("xxx");
        order.setReceiverMobile("13311112222");
        order.setCreateTime(new Date());
        order.setCreateUser("xxx");
        order.setUpdateTime(new Date());
        order.setUpdateUser("xxx");
        orderMapper.insertSelective(order);

        OrderItem orderItem = new OrderItem();
        orderItem.setOrderId(order.getId());
        orderItem.setProductId(product.getId());
        orderItem.setPurchasePrice(product.getPrice());
        orderItem.setPurchaseNum(purchaseProductNum);
        orderItem.setCreateUser("xxx");
        orderItem.setCreateTime(new Date());
        orderItem.setUpdateTime(new Date());
        orderItem.setUpdateUser("xxx");
        orderItemMapper.insertSelective(orderItem);
        platformTransactionManager.commit(transaction);
        return order.getId();
    }
```

# Distributed lock
## Use cases
* Efficiency: Taking a lock saves you from unnecessarily doing the same work twice (e.g. some expensive computation).  
	- e.g. If the lock fails and two nodes end up doing the same piece of work, the result is a minor increase in cost (you end up paying 5 cents more to AWS than you otherwise would have)
	- e.g. SNS scenarios: A minor inconvenience (e.g. a user ends up getting the same email notification twice).
	- e.g. eCommerce website inventory control

* Correctness: Taking a lock prevents concurrent processes from stepping on each others’ toes and messing up the state of your system. If the lock fails and two nodes concurrently work on the same piece of data, the result is a corrupted file, data loss, permanent inconsistency, the wrong dose of a drug administered to a patient, or some other serious problem.

## Requirementss
* Exclusive
* Avoid deadlock
* High available
* Reentrant

## CP model
### Comparison
* [Comparison](https://developpaper.com/talking-about-several-ways-of-using-distributed-locks-redis-zookeeper-database/) between different ways to implement distributed lock
    - From the perspective of understanding difficulty (from low to high)
        - Database > Caching > Zookeeper
    - From the perspective of complexity of implementation (from low to high)
        - Zookeeper > Cache > Database
    - From a performance perspective (from high to low)
        - Cache > Zookeeper > = database
    - From the point of view of reliability (from high to low)
        - Zookeeper > Cache > Database


### Database
#### Ideas behind the scene
* Use database locks
	- Table lock 
	- Unique index

#### Approach
* Create a row within database. When there are multiple requests against the same record, only one will succeed. 

```
SELECT stock FROM tb_product where product_id=#{product_id};
UPDATE tb_product SET stock=stock-#{num} WHERE product_id=#{product_id} AND stock=#{stock};
```

#### Pros and Cons
* Suitable for low concurrency scenarios
* Low performance because needs to access database

### Zookeeper
* Please see the distributed lock section in [Zookeeper](https://github.com/DreamOfTheRedChamber/system-design/blob/master/zookeeper.md)


### etcd
#### Operations
1. business logic layer apply for lock by providing (key, ttl)
2. etcd will generate uuid, and write (key, uuid, ttl) into etcd
3. etcd will check whether the key already exist. If no, then write it inside. 
4. After getting the lock, the heartbeat thread starts and heartbeat duration is ttl/3. It will compare and swap uuid to refresh lock

```
// acquire lock
curl http://127.0.0.1:2379/v2/keys/foo -XPUT -d value=bar -d ttl=5 prevExist=false

// renew lock based on CAS
curl http://127.0.0.1；2379/v2/keys/foo?prevValue=prev_uuid -XPUT -d ttl=5 -d refresh=true -d prevExist=true

// delete lock
curl http://10.10.0.21:2379/v2/keys/foo?prevValue=prev_uuid -XDELETE
```

## AP model - Redis

### Internals
* Distributed lock needs to serialize the processing of different events. Redis is based on a single thread, which serialize different events in nature. 

### Efficiency implementation
#### Acquire lock
* Before Redis version 2.6.12, set and expire are two separate commands
	- Deadlock if SETNX succeed but EXPIRE fails

```
// return 1 if success; return 0 otherwise
SETNX Key Value   (key=lock id, value=currentTime + timeout)

// set expiration time
EXPIRE Key seconds

// Execute multiple commands
MULTI 
EXEC
```

* Additional parameters could be passed to redis SET command (version 2.6.12)
	- SET resource_name my_random_value NX PX value

#### Release lock
* DELETE

#### Pros
* Lock is stored in memory. No need to access disk

#### Cons
##### Limited use cases
* Only applicable for efficiency use cases, not for correctness use cases.
	+ Efficiency
		- You could use a single Redis instance, of course you will drop some locks if the power suddenly goes out on your Redis node, or something else goes wrong. But if you’re only using the locks as an efficiency optimization, and the crashes don’t happen too often, that’s no big deal. This “no big deal” scenario is where Redis shines. At least if you’re relying on a single Redis instance, it is clear to everyone who looks at the system that the locks are approximate, and only to be used for non-critical purposes.
		- Add on top of the single application case, you could use master-slave setup for high availability. 
	+ Correctness
		- A simple master - slave setup won't work. Think about the following scenario: 
			1. Client A writes an entry A to master. 
			2. Master dies before the asynchronous replication of the write operation reaches slave. 
			3. The slave becomes the master
			4. Client B writes the same entry A to original salve (current master)
			5. Now A and B share the same lock.
		- You will need to rely on Redlock. However, there are [some concerns](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) about it. To summarize:
			- Redlock does not have any facility to generate fencing tokens. And it is not straightforward to repurpose Redlock for generating fencing tokens. 
				- Relying on expiration time to avoid deadlock is not reliable. 
					- What if the lock owner dies? The lock will be held forever and we could be in a deadlock. To prevent this issue Redis will set an expiration time on the lock, so the lock will be auto-released. However, if the time expires before the task handled by the owner isn't yet finish, another microservice can acquire the lock, and both lock holders can now release the lock causing inconsistency. 
					- A fencing token needed to be used to avoid race conditions. Please see [this post](https://medium.com/@davidecerbo/everything-i-know-about-distributed-locks-2bf54de2df71) for details. 
			- Redlock depends on a lot of timing assumptions
				1. All Redis nodes hold keys for approximately the right length of time before expiring
				2. The network delay is small compared to the expiry duration
				3. Process pauses are much shorter than the expiry duration

##### Link to history on RedLock
* Typical failures causing [failures of distributed locks](https://redislabs.com/ebook/part-2-core-concepts/chapter-6-application-components-in-redis/6-2-distributed-locking/6-2-2-simple-locks/)
* What Redlock tries to solve?
    - The simplest way to use Redis to lock a resource is to create a key in an instance. The key is usually created with a limited time to live, using the Redis expires feature, so that eventually it will get released (property 2 in our list). When the client needs to release the resource, it deletes the key.
    - Superficially this works well, but there is a problem: this is a single point of failure in our architecture. What happens if the Redis master goes down? Well, let’s add a slave! And use it if the master is unavailable. This is unfortunately not viable. By doing so we can’t implement our safety property of mutual exclusion, because Redis replication is asynchronous.
    - There is an obvious race condition with this model:
        - Client A acquires the lock in the master.
        - The master crashes before the write to the key is transmitted to the slave.
        - The slave gets promoted to master.
        - Client B acquires the lock to the same resource A already holds a lock for. SAFETY VIOLATION!
* How to implement distributed lock with Redis, an algorithm called [RedLock](https://redis.io/topics/distlock)
    - How to implement it in a single instance case
    - How to extend the single instance algorithm to cluster
* [A hot debate on the security perspective of RedLock algorithm](http://zhangtielei.com/posts/blog-redlock-reasoning.html).


