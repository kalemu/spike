Spike项目介绍
项目介绍
模拟秒杀场景,参考以下视频完成：
https://www.bilibili.com/video/BV13C4y1s7Ry
项目用到的技术如下：
springboot
mybatis
redis
dubbo
zookeeper
druid
rabbitmq
项目目录如图所示：

spike-api 公用代码层 【提供者service接口、所有实体类、公用工具类】
spike-service 业务逻辑层 【与数据库进行交互】
spike-web  pc端web层 【使用RPC框架Dubbo远程调用业务逻辑层】
数据库表：
商品表：
CREATE TABLE `product` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `name` varchar(255) DEFAULT NULL COMMENT '商品名',
  `price` double(10,2) DEFAULT NULL COMMENT '价格',
  `stock` int(11) DEFAULT NULL COMMENT '库存',
  `pic` varchar(255) DEFAULT NULL COMMENT '图片路径',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8;

订单表：
CREATE TABLE `seckillorder` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '订单主键',
  `product_id` int(11) DEFAULT NULL COMMENT '商品主键',
  `amount` decimal(10,2) DEFAULT NULL COMMENT '总金额',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

Service层主要逻辑：
package com.spike.service.impl;

// @Service：暴露接口，注意是dubbo包下的注解。
@Service(timeout = 3000)
@Transactional
public class OrderServiceImpl implements OrderService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private ProductService productService;

    /**
     * 创建秒杀订单。
     * @param productId 商品Id
     */
    @Override
    public void secKill(Long productId) {

        // 查询商品信息
        Product product = productService.getProductById(productId);

        if (product.getStock() <= 0) {
            throw new RuntimeException("商品库存已卖完。");
        }

        // 减库存
        Integer updateStock = productService.decrProductStock(productId);
        // 返回0代表减库存失败
        if (updateStock <= 0) {
            throw new RuntimeException("商品库存已售完。");
        }

        // 创建秒杀订单
        Order order = new Order();
        order.setProductId(productId);
        order.setAmount(product.getPrice());
        saveOrder(order);
    }

    /**
     * 保存订单
     * @param order
     */
    @Override
    public void saveOrder(Order order) {
        orderMapper.insertSelective(order);
    }

}

Controller层主要逻辑：
package com.spike.controller;

@Controller
@RequestMapping("order")
@Slf4j
public class OrderController {

    @Reference
    private OrderService orderService;

    @Reference
    private ProductService productService;

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    /**
     * 多线程在“读操作大于写操作”的情况下使用ConcurrentHashMap
     */
    public static Map<Long, Boolean> concurrentHashMap = new ConcurrentHashMap<>();

    @Autowired
    private ZooKeeper zooKeeper;

    /**
     * 会在服务器加载Servlet的时候运行，并且只会被服务器执行一次。
     * 将数据库中所有商品信息存储到Redis中
     */
    @PostConstruct
    public void initRedisValue() {
        List<Product> listProducts = productService.getListProducts();
        // 循环遍历将数据库中所有商品信息存储到Redis中
        listProducts.forEach(product -> stringRedisTemplate.opsForValue().set(Constants.REDIS_PRODUCT_STOCK_PREFIX + product.getId(), product.getStock().toString()));
    }

    /**
     * 秒杀商品
     * @param productId 商品主键
     * @return
     */
    @PostMapping("secKill/{productId}")
    public ResponseEntity<Void> secKill(@PathVariable("productId") Long productId) throws KeeperException, InterruptedException {
        if (concurrentHashMap.get(productId) != null) {
            throw new RuntimeException("库存已售完，创建订单失败！");
        }

        // 使用 “原子加” 来减一 线程安全的减库存
        Long stock = stringRedisTemplate.opsForValue().increment(Constants.REDIS_PRODUCT_STOCK_PREFIX + productId, -1);
        if (stock < 0) {
            // 添加已售完标识
            concurrentHashMap.put(productId, true);
            System.out.println("添加标识id：" + productId);
            // 还原Redis库存为0，防止Mysql和Redis的数据不一致
            stringRedisTemplate.opsForValue().increment(Constants.REDIS_PRODUCT_STOCK_PREFIX + productId, 1);

            // 路径：/product_stock_flag/1
            String nodePath = Constants.getZookeeperProductStockPath(productId);
            if (zooKeeper.exists(nodePath, true) == null) {
                zooKeeper.create(nodePath, "true".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
            }
            // 监听该节点
            zooKeeper.exists(nodePath, true);
            throw new RuntimeException("创建订单失败  Redis");

        }

        try {
            orderService.secKill(productId);
        } catch (Exception e) {

            if (concurrentHashMap.get(productId) != null) {
                concurrentHashMap.remove(productId);
            }

            // 假设Redis减了库存但是数据库宕机了没减库存，需要将Redis的库存还原。
            stringRedisTemplate.opsForValue().increment(Constants.REDIS_PRODUCT_STOCK_PREFIX + productId, 1);

            String nodePath = Constants.getZookeeperProductStockPath(productId);
            // 当该商品已售完且在数据库发生异常时，才会进行修改。
            if (zooKeeper.exists(nodePath,true) != null) {
                // -1 代表不使用乐观锁
                zooKeeper.setData(nodePath, "false".getBytes(), -1);
            }
            
            // 重新监听该节点
            zooKeeper.exists(nodePath, true);
            log.error("创建订单失败", e);
            e.printStackTrace();
            // 库存卖完报 500
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
        }
        return ResponseEntity.ok().build();
    }


}

用jmeter测试： 100个线程，5个循环测试


防止超卖思路：
根据商品主键更新库存，利用到了数据库的行锁，多个线程同时发送一条sql，mysql会上行锁，让线程排队执行，执行成功后返回更新的结果，若返回0代表修改成功0条，说明库存已经清零了，若返回大于0 的数字说明更新成功了，接着我们就可以创建订单了。
# stock代表库存属性
update product set stock=stock-1 where stock > 0 and id = #{productId}


使用Redis优化思路：
不加Redis，用户访问后台，请求会直接打到Mysql上，在高并发情况下，会让数据库直接宕机，因为Mysql支持的并发数很小。
假设现在有很多用户同时在抢一个商品，但是到最后我们发现库存都清零了，还是有请求打到数据库，最后返回无库存，这样的请求我们需要做一个判断，使用Redis的缓存同步技术。
预先让数据库的库存保存到Redis中，用户发送请求到后台使用Redis中的“原子减”操作来使库存减1，并返回当前库存中的值，判断库存是否小于0，若小于0直接返回无库存，并让Redis的库存加1，目的是保持数据库与Redis数据的一致性，加上Redis的目的是减轻数据库的并发压力。Redis支持的并发数是Mysql的好几倍。
// 使用 “原子加” 来减一 线程安全的减库存
Long stock = stringRedisTemplate.opsForValue().increment(Constants.REDIS_PRODUCT_STOCK_PREFIX + productId, -1);
if (stock < 0) {
    // 还原Redis库存为0，防止Mysql和Redis的数据不一致
    stringRedisTemplate.opsForValue().increment(Constants.REDIS_PRODUCT_STOCK_PREFIX + productId, 1);
    throw new RuntimeException("商品已售完，创建订单失败。");
}


若数据库发生异常，我们将这个异常捕捉到，数据库会执行回滚操作，但Redis不会执行回滚操作，所以我们将Redis的库存加1，目的是保证数据库与Redis的数据一致性。
try {
    // 秒杀方法
    orderService.secKill(productId);
} catch (Exception e) {
    // 假设Redis减了库存但是数据库宕机了，数据库会进行回滚操作
    // 但Redis不会进行回滚，所以我们需手动将Redis的库存加1。
    stringRedisTemplate.opsForValue().increment(Constants.REDIS_PRODUCT_STOCK_PREFIX + productId, 1);
    log.error("创建订单失败", e);
}


使用JVM层面的优化思路：若商品已售完则给该商品添加一个标识
每次请求到后台的时候，先判断ConcurrentHashMap中是否有当前商品id，若有代表该商品已售完。
/**
 * 多线程在“读操作大于写操作”的情况下使用ConcurrentHashMap
 */
public static Map<Long, Boolean> concurrentHashMap = new ConcurrentHashMap<>();

/**
 * 秒杀商品
 * @param productId 商品主键
 * @return
 */
@PostMapping("secKill/{productId}")
public ResponseEntity<Void> secKill(@PathVariable("productId") Long productId) throws KeeperException, InterruptedException {
    if (concurrentHashMap.get(productId) != null) {
        throw new RuntimeException("库存已售完，创建订单失败！");
    }
        // 使用 “原子加” 来减一 线程安全的减库存
    Long stock = stringRedisTemplate.opsForValue().increment(Constants.REDIS_PRODUCT_STOCK_PREFIX + productId, -1);
    if (stock < 0) {
        // 添加已售完标识
        concurrentHashMap.put(productId, true);
        System.out.println("添加标识id：" + productId);
        // 还原Redis库存为0，防止Mysql和Redis的数据不一致
        stringRedisTemplate.opsForValue().increment(Constants.REDIS_PRODUCT_STOCK_PREFIX + productId, 1);
        throw new RuntimeException("商品已售完，创建订单失败。");
     }
}


若数据库发生异常，我们将这个异常捕捉到，数据库会执行回滚操作，我们判断ConcurrentHashMap中是否有对应的商品，若有的话，删除即可，因为数据库发生回滚，代表数据库的库存没有减成功，此时如果ConcurrentHashMap中有已售完的标识的话则会直接返回已售完，但是这并不是我们想要的结果，所以需要将已售完标记删除。
try {
    orderService.secKill(productId);
} catch (Exception e) {
    // 判断当前商品是否有已售完标记，如果有则删除
    if (concurrentHashMap.get(productId) != null) {
        concurrentHashMap.remove(productId);
    }

    // 假设Redis减了库存但是数据库宕机了，数据库会进行回滚操作
   // 但Redis不会进行回滚，所以我们需手动将Redis的库存加1。
    stringRedisTemplate.opsForValue().increment(Constants.REDIS_PRODUCT_STOCK_PREFIX + productId, 1);
    log.error("创建订单失败", e);
}


使用Zookeeper让多个web服务器同步JVM标识思路：
假设有两台服务器，web-8080与web-8081，启动服务会在Zookeeper中创建一个商品库存节点 “/product_stock_flag” ，若访问web-8080服务器的后台，此时商品库存为0，那么下一个请求再访问，就会设置JVM层面的已售完标识，接着会查看Zookeeper当中是否有 “/product_stock_flag/商品ID” 的节点如果没有的话则创建，并让本服务监听该节点的数据内容变化
// 使用 “原子加” 来减一 线程安全的减库存
Long stock = stringRedisTemplate.opsForValue().increment(Constants.REDIS_PRODUCT_STOCK_PREFIX + productId, -1);
if (stock < 0) {
    // 添加已售完标识
    concurrentHashMap.put(productId, true);
    System.out.println("添加标识id：" + productId);
    // 还原Redis库存为0，防止Mysql和Redis的数据不一致
    stringRedisTemplate.opsForValue().increment(Constants.REDIS_PRODUCT_STOCK_PREFIX + productId, 1);

    // 路径：/product_stock_flag/1
    String nodePath = Constants.getZookeeperProductStockPath(productId);
    if (zooKeeper.exists(nodePath, true) == null) {
        zooKeeper.create(nodePath, "true".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
    }
    // 监听该节点
    zooKeeper.exists(nodePath, true);
    throw new RuntimeException("创建订单失败  Redis");

}

当数据库抛出异常了，会将该节点的值变为“false”，因为我们监听了该节点的变化，所以会通过回调函数来将 ConcurrentHashMap 中 key 为商品 id 的数据给删除掉。
try {
    orderService.secKill(productId);
} catch (Exception e) {

    if (concurrentHashMap.get(productId) != null) {
        concurrentHashMap.remove(productId);
    }

    // 假设Redis减了库存但是数据库宕机了，数据库会进行回滚操作，但Redis不会进行回滚，所以我们需手动将Redis的库存加1。
    stringRedisTemplate.opsForValue().increment(Constants.REDIS_PRODUCT_STOCK_PREFIX + productId, 1);

    String nodePath = Constants.getZookeeperProductStockPath(productId);
    // 当该商品已售完且在数据库发生异常时，才会进行修改。
    if (zooKeeper.exists(nodePath,true) != null) {
        // -1 代表不使用乐观锁
        zooKeeper.setData(nodePath, "false".getBytes(), -1);
    }
    // 重新监听该节点
    zooKeeper.exists(nodePath, true);
    log.error("创建订单失败", e);
    e.printStackTrace();
    // 库存卖完报 500
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
}


Zookeeper监听器回调函数：

@Bean
public ZooKeeper zookeeperConnectionConfig(){
    try {
        CountDownLatch countDownLatch = new CountDownLatch(1);
        zooKeeper = new ZooKeeper("192.168.20.255:2181", 5000, watchedEvent -> {
            if (watchedEvent.getType() == Watcher.Event.EventType.None) {
                if (watchedEvent.getState() == Watcher.Event.KeeperState.SyncConnected) {
                    System.out.println("连接成功");
                    try {
                        if (zooKeeper.exists(Constants.ZOOKEEPER_PRODUCT_STOCK_FLAG_PREFIX,false) == null) {
                            zooKeeper.create(Constants.ZOOKEEPER_PRODUCT_STOCK_FLAG_PREFIX,"true".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE,CreateMode.PERSISTENT);
                        }
                    } catch (KeeperException | InterruptedException e) {
                        e.printStackTrace();
                    }
                    countDownLatch.countDown();
                }

            }
            System.out.println("进入Zookeeper监听器");
            // 当Zookeeper中的节点发生改变时
            if (watchedEvent.getType() == Watcher.Event.EventType.NodeDataChanged) {
                // 将路径截取下来
                try {
                    String nodeData = new String(zooKeeper.getData(watchedEvent.getPath(), false, null));
                    if ("false".equals(nodeData)) {
                        String nodePath = watchedEvent.getPath().substring(Constants.ZOOKEEPER_PRODUCT_STOCK_FLAG_PREFIX.length() + 1);
                        System.out.println("进入到判断当中 id："+nodePath+", 数据为："+ nodeData);
                        // ConcurrentHashMap使用键删除的话，必须将变量转换成泛型对应的类型才可以删除掉！！
                        Long productId = Long.valueOf(nodePath);
                        if (OrderController.concurrentHashMap.get(productId) != null) {
                            OrderController.concurrentHashMap.remove(productId);
                        }
                        System.out.println("更新JVM层面的标识");
                        System.out.println(OrderController.concurrentHashMap);
                        System.out.println("还原库存成功");
                    }
                } catch (KeeperException | InterruptedException e) {
                    e.printStackTrace();
                }

            }
        });
        countDownLatch.await();
    } catch (InterruptedException | IOException e) {
        System.out.println("Zookeeper抛出异常");
        e.printStackTrace();
    }
    return zooKeeper;
}

