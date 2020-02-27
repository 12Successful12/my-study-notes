# 一、SpringBoot 与缓存

## 1、搭建基本环境

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>2.1.1</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
```

1. 导入数据库文件，创建出 department和employee 表

2. 在 bean 包下，创建 javaBean 封装数据

3. 整合 MyBatis 操作数据库

   3.1 配置数据源信息

   ```properties
   spring.datasource.url=jdbc:mysql://localhost:3306/web01?serverTimezone=GMT
   spring.datasource.username=root
   spring.datasource.password=12345
   #spring.datasource.driver-class-name 可以不用配，因为在spring.datasource.url中会自动配置。
   
   #	打印SQL语句
   logging.level.com.hzj.mapper=debug
   ```

   3.2 使用注解版的 MyBatis（@MapperScan()指定需要扫描的mapper接口所在的包）

   ```java
   package com.hzj.mapper;
   
   import com.hzj.bean.Employee;
   import org.apache.ibatis.annotations.Delete;
   import org.apache.ibatis.annotations.Insert;
   import org.apache.ibatis.annotations.Select;
   import org.apache.ibatis.annotations.Update;
   
   public interface EmployeeMapper {
   
       @Select("select * from employee where id = #{id}")
       public Employee getEmpById(Integer id);
   
       @Update("update employee set lastName=#{lastName},email=#{email},gender=#{gender},d_id=#{dId} where id=#{id}")
       public void updateEmp(Employee employee);
   
       @Delete("delete from employee where id=#{id}")
       public void deleteEmpById(Integer id);
   
       @Insert("insert into employee(lastName,email,gender,d_id) values(#{lastName},#{email},#{gender},#{dId})")
       public void insertEmp(Employee employee);
   }
   ```

4. 写 EmployeeService，略

5. 写 EmployeeController，略

## 2、快速体验缓存

1. 开启基于注解的缓存	**@EnableCaching**

2. 标注缓存注解即可：

   **@Cacheable**

   **@CacheEvict**

   **@CachePut**

### 2.1 @EnableCaching

该注解标注在 主启动类上

```java
@MapperScan("com.hzj.mapper")
@SpringBootApplication
@EnableCaching		//	开启基于注解的缓存
public class Springboot7CacheApplication {
    public static void main(String[] args) {
        SpringApplication.run(Springboot7CacheApplication.class, args);
    }
}
```

### 2.2 @Cacheable

将方法的运行结果进行缓存，以后再要相同的数据，直接从缓存中获取，不用调用方法；

CacheManager 管理多个 Cache 组件，对缓存的真正 CRUD 操作在 Cache 组件中，每一个缓存组件有自己唯一一个名字。

+ 注意：在主启动类上加了@EnableCaching注解后，若在业务层只加上 @Cacheable，不加属性，那么会报错：

```properties
No cache could be resolved for 'Builder[public com.hzj.bean.Employee com.hzj.service.EmployeeService.getEmp(java.lang.Integer)] caches=[] | key='' | keyGenerator='' | cacheManager='' | cacheResolver='' | condition='' | unless='' | sync='false'' using resolver 'org.springframework.cache.interceptor.SimpleCacheResolver@606161d9'. At least one cache should be provided per cache operation.
```

#### 2.2.1 原理 ：

1. 自动配置类：CacheAutoConfiguration
2. 缓存的配置类：

```properties
org.springframework.boot.autoconfigure.cache.GenericCacheConfiguration
org.springframework.boot.autoconfigure.cache.JCacheCacheConfiguration
org.springframework.boot.autoconfigure.cache.EhCacheCacheConfiguration
org.springframework.boot.autoconfigure.cache.HazelcastCacheConfiguration
org.springframework.boot.autoconfigure.cache.InfinispanCacheConfiguration
org.springframework.boot.autoconfigure.cache.CouchbaseCacheConfiguration
org.springframework.boot.autoconfigure.cache.RedisCacheConfiguration
org.springframework.boot.autoconfigure.cache.CaffeineCacheConfiguration
org.springframework.boot.autoconfigure.cache.SimpleCacheConfiguration
org.springframework.boot.autoconfigure.cache.NoOpCacheConfiguration
```

3. 哪个配置类默认生效？

   **SimpleCacheConfiguration**

4. **SimpleCacheConfiguration**给容器中注册了一个 CacheManager：ConcurrentMapCacheManager

```java
@Configuration(
    proxyBeanMethods = false
)
@ConditionalOnMissingBean({CacheManager.class})
@Conditional({CacheCondition.class})
class SimpleCacheConfiguration {
    SimpleCacheConfiguration() {
    }

    @Bean
    ConcurrentMapCacheManager cacheManager(CacheProperties cacheProperties, CacheManagerCustomizers cacheManagerCustomizers) {
        ConcurrentMapCacheManager cacheManager = new ConcurrentMapCacheManager();
        List<String> cacheNames = cacheProperties.getCacheNames();
        if (!cacheNames.isEmpty()) {
            cacheManager.setCacheNames(cacheNames);
        }

        return (ConcurrentMapCacheManager)cacheManagerCustomizers.customize(cacheManager);
    }
}
```

5. ConcurrentMapCacheManager 可以获取和创建 ConcurrentMapCache 类型的缓存组件；ConcurrentMapCache它的作用是将数据保存在 ConcurrentMap 中。

#### 2.2.2 运行流程：

+ @Cacheable：

1. 方法运行之前，先去查询 Cache（缓存组件），按照 cacheNames 指定的名字获取；

   （CacheManager先获取相应的缓存），第一次获取缓存如果没有 Cache 组件，会自动创建。

2. 去 Cache 中查找缓存的内容，使用一个 key，默认就是方法的参数；

   key是按照某种策略生成的 ，默认是使用 keyGenerator生成的，默认使用SimpleKeyGenerator 生成 key。

   ​	SimpleKeyGenerator 生成 key 的默认策略：

   ​		如果没有参数：	key = new SimpleKey()；

   ​		如果有一个参数：key = 参数的值；

   ​		如果有多个参数：key = new SimpleKey(params)；

3. 没有查到缓存就调用目标方法。

4. 将目标方法返回的结果，放进缓存中。

+ 总结：@Cacheable 标注的方法执行之前，先来检查缓存中有没有这个数据，默认按照参数的值作为 key去查询缓存，如果没有就运行方法并将结果放入缓存。以后再来调用就可以直接使用缓存中的数据。
+ 核心：
  + 使用 CacheManager【ConcurrentMapCacheManager】按照名字得到 Cache【ConcurrentMapCache】 组件；
  + key 使用 keyGenerator生成的，默认是 SimpleKeyGenerator； 

#### 2.2.3 @Cacheable属性

+ 属性：
  + cacheNames/value：指定缓存组件的名字；将方法的返回结果放在哪个缓存中，是数组的方式，可以指定多个缓存。
  + key：缓存数据使用的key，可以用它来指定。默认是使用方法参数的值。   **`方法参数-方法返回值`**  。
    + 可以编写 SpEL：	#id；(参数id的值)	等价于：#a0、#p0、#root.args[0]。
  + keyGenerator：key的生成器；可以自己指定 key 的生成器的组件 id。
    + key/keyGenerator：二选一使用。
  + cacheManager：指定缓存管理器；或者 cacheResolver：指定获取解析器。
  + condition：指定符合条件的情况下才缓存。
    + <font color="blue">condition = “#id>0”</font>
  + unless：否定缓存；当 unless 指定的条件为 true，方法的返回值就不会被缓存；可以获取到结果进行判断。
    + <font color="blue">unless = "#result == null "</font>
  + sync：是否使用异步模式。

--------------------------------------------------

+ 1、cacheNames/value：

```java
@Service
public class EmployeeService {
    @Autowired
    private EmployeeMapper employeeMapper;

//    @Cacheable(cacheNames = {"emp"})
//    @Cacheable(value = {"emp"})
    public Employee getEmp(Integer id){
        return employeeMapper.getEmpById(id);
    }
}
```

+ 2、key：

​																		**SpEL：**

![](https://github.com/12Successful12/my-study-notes/blob/master/springboot-study-notes/images/1.png)

```java
@Service
public class EmployeeService {
    @Autowired
    private EmployeeMapper employeeMapper;
    
    // #root.methodName+'['+id+']'  等价于 getEmp[id]
    @Cacheable(cacheNames = {"emp"},key = "#root.methodName+'['+id+']'")
    public Employee getEmp(Integer id){
        return employeeMapper.getEmpById(id);
    }
}
```

+ 3、keyGenerator

```java
@Service
public class EmployeeService {

    @Autowired
    private EmployeeMapper employeeMapper;

    @Cacheable(cacheNames = {"emp"},keyGenerator = "myKeyGenerator")
    public Employee getEmp(Integer id){
        System.out.println("查询 "+id+" 号员工");
        return employeeMapper.getEmpById(id);
    }
}
```

```java
@Configuration
public class MyCacheConfig {
    @Bean("myKeyGenerator")
    public KeyGenerator keyGenerator(){
        return new KeyGenerator() {
            @Override
            public Object generate(Object o, Method method, Object... objects) {
                return method.getName()+"["+ Arrays.asList(objects).toString()+"]";
            }
        };
    }
}
```

+ 4、condition：

```java
@Service
public class EmployeeService {

    @Autowired
    private EmployeeMapper employeeMapper;

    @Cacheable(cacheNames = {"emp"},keyGenerator = "myKeyGenerator",condition = "#a0 > 1")	
    public Employee getEmp(Integer id){
        System.out.println("查询 "+id+" 号员工");
        return employeeMapper.getEmpById(id);
    }
}
```

condition = "#a0 > 1"：第一个参数的值 >1 的时候才进行缓存。

+ 5、unless：

```java
@Service
public class EmployeeService {

    @Autowired
    private EmployeeMapper employeeMapper;

    @Cacheable(cacheNames = {"emp"},keyGenerator = "myKeyGenerator",condition = "#a0 > 1",unless = "#id == 2")
    public Employee getEmp(Integer id){
        System.out.println("查询 "+id+" 号员工");
        return employeeMapper.getEmpById(id);
    }
}
```

unless = "#id == 2"：如果第一个参数的值是 2，结果不缓存。

+ 6、sync：

```java
java.lang.IllegalStateException: @Cacheable(sync=true) does not support unless attribute on 'Builder
```

所以，要想使用 sync，就不能使用 unless。



### 2.3 @CachePut

+ @CachePut：既调用方法，又更新缓存数据；同步更新缓存。
+ 修改了数据库的某个数据，同时更新缓存。

#### 2.3.1 运行时机：

+ 先调用目标方法；
+ 将目标方法的结果缓存起来

#### 2.3.2 测试步骤：

```java
@Service
public class EmployeeService {

    @Autowired
    private EmployeeMapper employeeMapper;

    @Cacheable(cacheNames = {"emp"})
    public Employee getEmp(Integer id){
        System.out.println("查询 "+id+" 号员工");
        return employeeMapper.getEmpById(id);
    }

    @CachePut(value = "emp")
    public Employee updateEmp(Employee employee){
        System.out.println("updateEmp "+employee);
        employeeMapper.updateEmp(employee);
        return employee;
    }
}
```

1. 查询 1号员工，查到的结果会放在缓存中；

   ​		<font color="orange">key：1	value：lastName=a</font>

2. 以后查询还是之前的结果；

3. 更新 1号员工；【lastName = zhangsan】

   ​		<font color="orange">key：传入的 employee对象	value：返回的 employee对象</font>

4. 再次查询 1号员工

   按道理这次查询结果应该是更新后的员工信息，**但是，**查询到的结果是**没更新前的**；

   为什么会这样？<font color="orange">因为 1号员工 没有在缓存中更新！</font>

   **解决办法：统一指定 key。**

   ​	key = "#employee.id" ：使用传入的参数的员工 id；

   ​	key = "#result.id"		：使用返回后的 id；

   ​			@Cacheable 的 key 不能用 #result，因为@Cacheable在运行之前，就需要获取 key。

```java
@Service
public class EmployeeService {

    @Autowired
    private EmployeeMapper employeeMapper;

    @Cacheable(cacheNames = {"emp"})
    public Employee getEmp(Integer id){
        System.out.println("查询 "+id+" 号员工");
        return employeeMapper.getEmpById(id);
    }
		//	添加 key = "#employee.id" 或 key = "#result.id"
    @CachePut(value = "emp",key = "#employee.id")
    public Employee updateEmp(Employee employee){
        System.out.println("updateEmp "+employee);
        employeeMapper.updateEmp(employee);
        return employee;
    }
}
```

+ 统一 key 后，效果：
+ 先查询：

![](images\2.png)

+ 然后修改：

![](images\3.png)

+ 再次查询：

![](images\4.png)

#### 2.3.3 @CachePut属性

同 @Cacheable属性

### 2.4 @CacheEvict

+ @CacheEvict：缓存清除，一般与 delete一起用。

#### 2.4.1 测试步骤：

1. 先在数据库中查询 1号员工并放入缓存中
2. 然后删除 1号员工
3. 再次查询 1号员工
4. 可以发现是从数据库中查询的。

```java
@Service
public class EmployeeService {

    @Autowired
    private EmployeeMapper employeeMapper;

    @Cacheable(cacheNames = {"emp"})
    public Employee getEmp(Integer id){
        System.out.println("查询 "+id+" 号员工");
        return employeeMapper.getEmpById(id);
    }

    @CachePut(value = "emp",key = "#employee.id")
    public Employee updateEmp(Employee employee){
        System.out.println("updateEmp "+employee);
        employeeMapper.updateEmp(employee);
        return employee;
    }

    @CacheEvict(value = "emp",allEntries = true)
    public void deleteEmp(Integer id){
        System.out.println("deleteEmp "+id);
//        employeeMapper.deleteEmpById(id);
    }
}
```

#### 2.4.2 @CacheEvict属性

同 @Cacheable。

+ key：指定要清除的数据；

+ allEntries = true：指定清除这个缓存中所有的数据；

+ beforeInvocation = false：缓存的清除是否在方法之前执行。

  ​	默认代表 缓存清除操作 是在方法执行之后执行；如果出现异常，缓存就不会清除。

+ beforeInvocation = true：缓存的清除**是在**方法运行之前执行。无论方法是否出现异常，缓存都清除。



### 2.5 @Caching

+ @Caching 可以定义复杂的 缓存规则

```java
@Service
public class EmployeeService {

    @Autowired
    private EmployeeMapper employeeMapper;

    @Cacheable(cacheNames = {"emp"})
    public Employee getEmp(Integer id){
        System.out.println("查询 "+id+" 号员工");
        return employeeMapper.getEmpById(id);
    }

    @CachePut(value = "emp",key = "#employee.id")
    public Employee updateEmp(Employee employee){
        System.out.println("updateEmp "+employee);
        employeeMapper.updateEmp(employee);
        return employee;
    }


    @CacheEvict(value = "emp",allEntries = true)
    public void deleteEmp(Integer id){
        System.out.println("deleteEmp "+id);
//        employeeMapper.deleteEmpById(id);
    }

    @Caching(
            cacheable = {
                    @Cacheable(value = "emp",key = "#lastName")
             },
            put = {
                    @CachePut(value = "emp", key = "#result.id"),
                    @CachePut(value = "emp", key = "#result.email")
            }
    )
    public Employee getEmpByLastName(String lastName){
        return employeeMapper.getEmpByLastName(lastName);
    }
}
```

### 2.6 @CacheConfig

+ 若每个方法的 value/cacheNames 都一样，可以在**类上**标注 @CacheConfig

```java
@CacheConfig(cacheNames = "emp")    //  抽取缓存的公共配置
@Service
public class EmployeeService {

    @Autowired
    private EmployeeMapper employeeMapper;

    @Cacheable(/*cacheNames = {"emp"}*/)
    public Employee getEmp(Integer id){
        System.out.println("查询 "+id+" 号员工");
        return employeeMapper.getEmpById(id);
    }

    @CachePut(/*value = "emp",*/key = "#employee.id")
    public Employee updateEmp(Employee employee){
        System.out.println("updateEmp "+employee);
        employeeMapper.updateEmp(employee);
        return employee;
    }


    @CacheEvict(/*value = "emp",*/allEntries = true)
    public void deleteEmp(Integer id){
        System.out.println("deleteEmp "+id);
//        employeeMapper.deleteEmpById(id);
    }

    @Caching(
            cacheable = {
                    @Cacheable(/*value = "emp",*/key = "#lastName")
             },
            put = {
                    @CachePut(/*value = "emp",*/ key = "#result.id"),
                    @CachePut(/*value = "emp",*/ key = "#result.email")
            }
    )
    public Employee getEmpByLastName(String lastName){
        return employeeMapper.getEmpByLastName(lastName);
    }
}
```



## 3、整合 redis 作为缓存

Redis 是一个开源（BSD许可）的，内存中的数据结构存储系统，它可用作数据库、缓存和消息中间件。

### 3.1 安装 redis

使用 docker：

```shell
root@iZwz9bdw1ibdoiewowwmk9Z:/etc/docker# docker pull redis
root@iZwz9bdw1ibdoiewowwmk9Z:/etc/docker# docker images
root@iZwz9bdw1ibdoiewowwmk9Z:/etc/docker# docker run -d -p 6379:6379 --name=myredis redis
```

### 3.2 引入redis的starter

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
```

### 3.3 配置redis

```properties
spring.redis.host=120.25.164.172
```

### 3.4 Test测试redis操作数据

+ Redis 常见的五大数据类型
  + String（字符串）、List（列表）、Set（集合）、Hash（散列）、ZSet（有序集合）
+ stringRedisTemplate.opsForValue()【String（字符串）】
+ stringRedisTemplate.opsForList()【List（列表）】
+ stringRedisTemplate.opsForSet()【Set（集合）】
+ stringRedisTemplate.opsForHash()【Hash（散列）】
+ stringRedisTemplate.opsForZSet()【ZSet（有序集合）】
  + redisTemplate 也有以上 5种 方法。将前缀换成 redisTemplate 即可。

```java
    @Autowired  // 操作k-v都是字符串的
    private StringRedisTemplate stringRedisTemplate;

    @Autowired  //  k-v 都是对象的
    private RedisTemplate redisTemplate;
```



#### 3.4.1 序列化

```java
@SpringBootTest
class Springboot7CacheApplicationTests {

    @Autowired
    private EmployeeMapper employeeMapper;

    @Autowired  // 操作k-v都是字符串的
    private StringRedisTemplate stringRedisTemplate;

    @Autowired  //  k-v 都是对象的
    private RedisTemplate redisTemplate;

    @Test
    public void test1(){
        Employee empById = employeeMapper.getEmpById(1);
     //   默认如果保存对象，使用 JDK 序列化机制，序列化后的数据保存到 redis 中。
        redisTemplate.opsForValue().set("emp",empById);
    }
}
```

实体类需要 **implements Serializable**，否则，会报错。

**implements Serializable**运行程序后，

![](\images\5.png)

在数据的前后会增加序列化的值，解决办法：创建一个 MyRedisConfig 类，自己实现序列化：

+ MyRedisConfig .java

```java
package com.hzj.config;

import com.hzj.bean.Employee;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;

import java.net.UnknownHostException;

@Configuration
public class MyRedisConfig {

    @Bean
    public RedisTemplate<Object, Employee> empRedisTemplate(
            RedisConnectionFactory redisConnectionFactory)
            throws UnknownHostException {
        RedisTemplate<Object,Employee> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        Jackson2JsonRedisSerializer<Employee> serializer = new Jackson2JsonRedisSerializer<Employee>(Employee.class);
        template.setDefaultSerializer(serializer);
        return template;
    }
}
/*
写出这个代码的流程：
1、因为 SpringBoot 基于自动配置，所以查找 RedisAutoConfiguration
2、找到源码，二选一
    @Bean
    @ConditionalOnMissingBean(
        name = {"redisTemplate"}
    )
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) throws UnknownHostException {
        RedisTemplate<Object, Object> template = new RedisTemplate();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }

    @Bean
    @ConditionalOnMissingBean
    public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory) throws UnknownHostException {
        StringRedisTemplate template = new StringRedisTemplate();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }
3、因为 new StringRedisTemplate() 是用于简化 new RedisTemplate() 的，因此直接查看 new RedisTemplate()源码可以发现：
if (this.defaultSerializer == null) {
	this.defaultSerializer = new JdkSerializationRedisSerializer(this.classLoader != null ? this.classLoader : this.getClass().getClassLoader());
}			默认是 jdk 序列化
4、改写：
    @Bean
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) throws UnknownHostException {
        RedisTemplate<Object, Object> template = new RedisTemplate();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }
因为不想使用默认的序列化机制，因此需要放入一个 序列化器。选择 json格式的序列化器：Jackson2JsonRedisSerializer<T>
*/
```

+ Test

```java
@SpringBootTest
class Springboot7CacheApplicationTests {

    @Autowired
    private EmployeeMapper employeeMapper;

    @Autowired  // 操作k-v都是字符串的
    private StringRedisTemplate stringRedisTemplate;

    @Autowired  //  k-v 都是对象的
    private RedisTemplate redisTemplate;

    @Autowired
    RedisTemplate<Object, Employee> empRedisTemplate;
    
    @Test
    public void test1(){
        Employee empById = employeeMapper.getEmpById(1);
//        默认如果保存对象，使用 JDK 序列化机制，序列化后的数据保存到 redis 中。
//        redisTemplate.opsForValue().set("emp",empById);

//        1、将数据以 json 的方式保存
//          （1）自己将对象转为 json
//          （2）redisTemplate 默认的序列化规则
        empRedisTemplate.opsForValue().set("emp",empById);

    }
}
```

+ 效果图：

![](images\6.png)

### 3.5 测试缓存

+ 原理：CacheManager === Cache 缓存组件来实际给缓存中存取数据

  + 1）引入 redis 的 starter，容器中保存的是 RedisCacheManager；
  + 2）RedisCacheManager  帮我们创建 RedisCache 来作为缓存组件；RedisCache 通过操作 redis 缓存数据的；
  + 3）默认保存数据 k-v 都是Object，利用 jdk序列化保存。那如何保存为 json？
    + 1、引入了 redis 的 starter，cacheManager 变为 RedisCacheManager；
    + 2、默认创建的 RedisCacheManager 操作 redis 的时候使用的是 RedisTemplate<Object,Object>
    + 3、RedisTemplate<Object,Object> 默认是使用 jdk 的序列化机制；
  + 4）自定义 CacheManager：

  ```java
  @Configuration
  public class MyRedisConfig {
  
      @Bean
      public RedisTemplate<Object, Employee> empRedisTemplate(
              RedisConnectionFactory redisConnectionFactory)
              throws UnknownHostException {
          RedisTemplate<Object,Employee> template = new RedisTemplate<>();
          template.setConnectionFactory(redisConnectionFactory);
          Jackson2JsonRedisSerializer<Employee> serializer = new Jackson2JsonRedisSerializer<Employee>(Employee.class);
          template.setDefaultSerializer(serializer);
          return template;
      }
  
      @Bean
      public RedisCacheManager employeeCacheManager(RedisTemplate<Object,Employee> empRedisTemplate) {									// 报错，说没有构造器
          RedisCacheManager cacheManager = new RedisCacheManager(empRedisTemplate);
          //  key 多了一个前缀。因为使用了前缀，默认会将 CacheName 作为 key 的前缀
          //	报错，没有这个方法。
          cacheManager.setUsePrefix(true);
          return cacheManager;
  
      }
  }
  ```


### 坑！

实现 DeptService 代码，代码和 EmployeeService 差不多，然后访问相应的 路径，第一次访问的时候，数据成功访问，并且 redis 中也有数据；第二次访问的时候，会报错。即：

+ 缓存的数据能存入 redis，第二次从缓存中查询就不能反序列化回来。

+ 因为 存的是 dept 的 json 数据，CacheManager 默认使用 RedisTemplate<Object,Employee> 操作 Redis。
+ 因此需要配置多个 RedisCacheManager。

```java
@Configuration
public class MyRedisConfig {

    @Bean
    public RedisTemplate<Object, Employee> empRedisTemplate(
            RedisConnectionFactory redisConnectionFactory)
            throws UnknownHostException {
        RedisTemplate<Object,Employee> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        Jackson2JsonRedisSerializer<Employee> serializer = new Jackson2JsonRedisSerializer<Employee>(Employee.class);
        template.setDefaultSerializer(serializer);
        return template;
    }

    @Primary//由于有多个CacheManager，因此需要指定一个CacheManager为 主要的默认的CacheManager！！
    @Bean
    public RedisCacheManager employeeCacheManager(RedisTemplate<Object,Employee> empRedisTemplate) {
        RedisCacheManager cacheManager = new RedisCacheManager(empRedisTemplate);
        //  key 多了一个前缀。因为使用了前缀，默认会将 CacheName 作为 key 的前缀
        cacheManager.setUsePrefix(true);
        return cacheManager;
    }

    @Bean
    public RedisTemplate<Object, Department> deptRedisTemplate(
            RedisConnectionFactory redisConnectionFactory)
            throws UnknownHostException {
        RedisTemplate<Object,Department> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        Jackson2JsonRedisSerializer<Department> serializer = new Jackson2JsonRedisSerializer<Department>(Department.class);
        template.setDefaultSerializer(serializer);
        return template;
    }

    @Bean
    public RedisCacheManager departmentCacheManager(RedisTemplate<Object,Department> deptRedisTemplate) {
        RedisCacheManager cacheManager = new RedisCacheManager(deptRedisTemplate);
        //  key 多了一个前缀。因为使用了前缀，默认会将 CacheName 作为 key 的前缀
        cacheManager.setUsePrefix(true);
        return cacheManager;
    }
    
    /*
    一般情况下，将 SpringBoot 自动配置的 cacheManager 作为默认的、主要的 cacheManager
        @Bean
    RedisCacheManager cacheManager(CacheProperties cacheProperties, CacheManagerCustomizers cacheManagerCustomizers, ObjectProvider<org.springframework.data.redis.cache.RedisCacheConfiguration> redisCacheConfiguration, ObjectProvider<RedisCacheManagerBuilderCustomizer> redisCacheManagerBuilderCustomizers, RedisConnectionFactory redisConnectionFactory, ResourceLoader resourceLoader) {
        RedisCacheManagerBuilder builder = RedisCacheManager.builder(redisConnectionFactory).cacheDefaults(this.determineConfiguration(cacheProperties, redisCacheConfiguration, resourceLoader.getClassLoader()));
        List<String> cacheNames = cacheProperties.getCacheNames();
        if (!cacheNames.isEmpty()) {
            builder.initialCacheNames(new LinkedHashSet(cacheNames));
        }

        redisCacheManagerBuilderCustomizers.orderedStream().forEach((customizer) -> {
            customizer.customize(builder);
        });
        return (RedisCacheManager)cacheManagerCustomizers.customize(builder.build());
    }
    */
}
```

### 3.6 编码时的使用

+ 在业务层中，对缓存进行 CRUD 操作：

```java
    @Qualifier("empCacheManager")
    @Autowired
    RedisCacheManager empCacheManager;

//使用缓存管理器得到缓存，进行 api 调用即可。
    public Employee getEmpById(Integer id){
        //  获取某个缓存
        Cache emp = empCacheManager.getCache("emp");
        emp.put("k1","v1");
        return null;
    }
```



# 二、SpringBoot 与任务

## 2.1 异步任务入门案例

+ AsyncService.java

```java
package com.hzj.service;

import org.springframework.stereotype.Service;

@Service
public class AsyncService {

    public void hello() {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("过了 3秒，数据处理中。。。");
    }
}
```

+ AsyncController.java

```java
package com.hzj.controller;

import com.hzj.service.AsyncService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class AsyncController {

    @Autowired
    private AsyncService asyncService;

    @GetMapping("/hello")
    public String hello(){
        asyncService.hello();
        return "success";
    }
}
```

+ Springboot8TaskApplication.java

```java
package com.hzj;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Springboot8TaskApplication {

    public static void main(String[] args) {
        SpringApplication.run(Springboot8TaskApplication.class, args);
    }
}
```



+ **手动改为多线程的方式，若每次都要编码，这就很麻烦了。可以在业务层上使用 @Async 注解，告诉 spring 这是一个异步方法**

  + ```java
    @Service
    public class AsyncService {
    
        @Async	//	告诉 spring 这是一个异步方法
        public void hello() {
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("过了 3秒，数据处理中。。。");
        }
    }
    ```

+ **@Async 注解要想起作用，需要在 主启动类 上加上 @EnableAsync 注解**

  + ```java
    @EnableAsync	//	开启异步注解功能
    @SpringBootApplication
    public class Springboot8TaskApplication {
    
        public static void main(String[] args) {
            SpringApplication.run(Springboot8TaskApplication.class, args);
        }
    }
    ```

+ **此时，项目启动之后，访问路径就会立马显示出来。**



## 2.2 定时任务入门案例

+ ScheduleService.java

```java
package com.hzj.service;

import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

@Service
public class ScheduleService {

    /*
    秒、分、时、日、月、星期
    */
    @Scheduled(cron = "* * * * * *")
    public void hello(){
        System.out.println("hello...");
    }
}
```



+ 主启动类：Springboot8TaskApplication.java
+ @EnableScheduling	：开启基于注解的定时任务

```java
package com.hzj;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.annotation.EnableScheduling;

@EnableAsync
@EnableScheduling	//	开启基于注解的定时任务
@SpringBootApplication
public class Springboot8TaskApplication {

    public static void main(String[] args) {
        SpringApplication.run(Springboot8TaskApplication.class, args);
    }
}
```



## 2.3 邮件任务入门案例

### 2.3.1 引入依赖

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-mail</artifactId>
        </dependency>
```

### 2.3.2 修改配置

首先要去邮箱里开通 POP3/SMTP服务、IMAP/SMTP服务、Exchange服务、CardDAV/CalDAV服务

![](images\7.png)

```properties
					#发送者的邮箱
spring.mail.username=534096094@qq.com	
					#发送者的密码，出于安全考虑，用授权码代替密码
spring.mail.password=gtstkoszjelabijb
				#SMTP服务器的地址
spring.mail.host=smtp.qq.com
		#	若不配置这个，会报错说需要安全链接 ssl
		#	额外的配置都在 spring.mail.properties.
spring.mail.properties.mail.smtp.ssl.enable=true
```

+ spring.mail.host：这是SMTP服务器的地址

![](images\8.png)

### 2.3.3 测试代码

```java
package com.hzj;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.io.File;

@SpringBootTest
class Springboot8TaskApplicationTests {

    @Autowired
    JavaMailSenderImpl mailSender;

    /**
     * 简单邮件发送
     */
    @Test
    void test1() {
        SimpleMailMessage message = new SimpleMailMessage();
        //  邮件设置
        message.setSubject("通知");
        message.setText("您被腾讯录用了");

        message.setTo("接收者邮箱");
        message.setFrom("发送者邮箱");
    }

    /**
     * 复杂邮件发送
     */
    public void test2() {
        //1、创建一个复杂的消息邮件
        MimeMessage mimeMessage = mailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(mimeMessage, true);

        //2、邮件设置
        helper.setSubject("通知");
        helper.setText("您被腾讯录用了");

        helper.setTo("接收者邮箱");
        helper.setFrom("发送者邮箱");

        //3、上传文件
        helper.addAttachment("1.jpg",new File(""));
        
        mailSender.send(mimeMessage);
    }
}
```



# 三、SpringBoot 与安全Shiro

## 3.1 导入依赖

```xml
        <dependency>
            <groupId>org.apache.shiro</groupId>
            <artifactId>shiro-spring</artifactId>
            <version>1.4.0</version>
        </dependency>
```

## 3.2 编写 thymeleaf 模板页面

+ **welcome.html**

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>欢迎页面</title>
    <style>

        <!--/* 由于不懂前端，于是就去网上搜了下如何将文字置于页面中间，并作略微调整。*/-->
        .toCenter {
            text-align: center; /*让div内部文字居中*/
            border-radius: 20px;
            width: 500px;
            height: 250px;
            margin: auto;
            position: absolute;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
        }
    </style>
</head>
<body>
<div class="toCenter">
    <h1 align="center">欢迎使用武林秘籍管理系统</h1>
    <h2 align="center">游客您好，如果想查看武林秘籍，<a th:href="@{/login}"> 请登录 </a> </h2>
</div>

</body>
</html>
```

+ **login.html**

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>登录页面</title>
    <link rel="stylesheet" th:href="@{https://cdn.bootcss.com/twitter-bootstrap/4.4.1/css/bootstrap.min.css}">
    <style>

        <!--/* 由于不懂前端，于是就去网上搜了下如何将文字置于页面中间，并作略微调整。*/-->
        .toCenter {
            text-align: center; /*让div内部文字居中*/
            border-radius: 20px;
            width: 500px;
            height: 300px;
            margin: auto;
            position: absolute;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
        }
    </style>
</head>
<body>
<div class="toCenter">
    <h1 align="center">欢迎登录武林秘籍管理系统</h1>
    <br>
    <div class="container">
        <h3 th:text="${msg}" style="color: red"></h3>
        <form class="form-horizontal" th:action="@{/loginForm}" method="post">
            <div class="row form-group">
                <span class="col-md-offset-3"></span>
                <label class=" col-sm-2 col-md-2 control-label" style="text-align: right">用户名</label>
                <div class="col-sm-6 col-md-6">
                    <input type="text" class="form-control"  name="username" placeholder="用户名">
                </div>
            </div>
            <div class="row form-group">
                <label class="col-sm-2 col-md-2 control-label" style="text-align: right">密码</label>
                <div class="col-sm-6 col-md-6">
                    <input type="password" class="form-control" name="password" placeholder="密码">
                </div>
            </div>
            <div class="row form-group">
                <div class="col-sm-offset-3 col-sm-6">
                    <div class="checkbox">
                        <label>
                            <input type="checkbox" name="rememberme" value="yes"> Remember me
                        </label>
                    </div>
                </div>
            </div>
            <div class="row form-group">
                <div class="col-sm-offset-3 col-sm-6">
                    <button type="submit" class="btn btn-success" value="登录">登录</button>
                    <button type="reset" class="btn btn-warning" value="重置">重置</button>
                </div>
            </div>
        </form>
    </div>
</div>
</body>
</html>
```

+ **main.html**

```java
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org" xmlns:shiro="http://www.w3.org/1999/xhtml">
<!--xmlns:shiro="http://www.w3.org/1999/xhtml"-->
<!--<html xmlns:th="http://www.thymeleaf.org"-->
<!--      xmlns:shiro="http://www.pollix.at/thymeleaf/shiro">-->
<head>
    <meta charset="UTF-8">
    <title>首页</title>
    <link rel="stylesheet" th:href="@{https://cdn.bootcss.com/twitter-bootstrap/4.4.1/css/bootstrap.min.css}">
    <style type="text/css">
        .center-block {
            display: block;
            margin-left: auto;
            margin-right: auto;
        }
    </style>
</head>
<body>
<div class="container">

<h1 align="center">欢迎使用武林秘籍管理系统</h1>
<h2 align="center"> <span shiro:principal="">游客</span>，您好！ <a href="#" th:href="@{/logout}">退出登录</a></h2>
    <br>
    <shiro:hasAnyRoles name="level1,level2,level3">
        <div class="row">
            <div class="list-group  col-md-offset-5 col-md-4 center-block" style="text-align: center">
                <h3 style="color: green">（Level1）普通武林秘籍</h3>
                <a href="#" th:href="@{/level1/1}" class="list-group-item">罗汉拳</a>
                <a href="#" th:href="@{/level1/2}" class="list-group-item">武当长拳</a>
                <a href="#" th:href="@{/level1/3}" class="list-group-item">全真剑法</a>
            </div>
        </div>
    </shiro:hasAnyRoles>
    <br>
    <shiro:hasAnyRoles name="level2,level3">
        <div class="row">
            <div class="list-group  col-md-offset-5 col-md-4 center-block" style="text-align: center">
                <h3 style="color: blue">（Level2）高级武林秘籍</h3>
                <a href="#" th:href="@{/level2/1}" class="list-group-item">太极拳</a>
                <a href="#" th:href="@{/level2/2}" class="list-group-item">七伤拳</a>
                <a href="#" th:href="@{/level2/3}" class="list-group-item">梯云纵</a>
            </div>
        </div>
    </shiro:hasAnyRoles>
    <br>
    <shiro:hasAnyRoles name="level3">
        <div class="row">
            <div class="list-group  col-md-offset-5 col-md-4 center-block" style="text-align: center">
                <h3 style="color: red">（Level3）绝世武林秘籍</h3>
                <a href="#" th:href="@{/level3/1}" class="list-group-item">葵花宝典</a>
                <a href="#" th:href="@{/level3/2}" class="list-group-item">龟派气功</a>
                <a href="#" th:href="@{/level3/3}" class="list-group-item">独孤九剑</a>
            </div>
        </div>
    </shiro:hasAnyRoles>
</div>

<script th:src="@{https://cdn.bootcss.com/twitter-bootstrap/4.4.1/js/bootstrap.min.js}"></script>
</body>
</html>
```

+ **levelx/x.html**

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Level1</title>
</head>
<body>
<h1>Level1-1</h1>
<a href="#" th:href="@{/}">返回welcome页面</a>
</body>
</html>
```



## 3.3 编写 User

+ 原本是打不算写 User 实体类的，但是由于在 UserController 中获取不到页面的 rememberme复选框 状态，于是就写了 User实体类解决这个问题。

```java
package com.hzj.shiro.pojo;

public class User {
    private String username;
    private String password;
    private String rememberme;

    public String getRememberme() {
        return rememberme;
    }

    public void setRememberme(String rememberme) {
        this.rememberme = rememberme;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```



## 3.3 编写 UserController

```java
package com.hzj.shiro.controller;

import com.hzj.shiro.pojo.User;
import org.apache.shiro.SecurityUtils;
import org.apache.shiro.authc.AuthenticationException;
import org.apache.shiro.authc.UsernamePasswordToken;
import org.apache.shiro.subject.Subject;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;

@Controller
public class UserController {

    /**
     * 欢迎页面映射
     * @return
     */
    @GetMapping("/")
    public String welcome(){
        return "welcome";
    }

    /**
     * 登录页面映射
     * @return
     */
    @RequestMapping("/login")
    public String login() {
        return "login";
    }

    @GetMapping("/user/main")
    public String main() {
        return "pages/main";
    }

    /**
     * （Level1）普通武林秘籍映射
     * @param path
     * @return
     */
    @GetMapping("/level1/{path}")
    public String level1(@PathVariable("path") String path){
        return "pages/level1/"+path;
    }

    /**
     * （Level2）高级武林秘籍
     * @param path
     * @return
     */
    @GetMapping("/level2/{path}")
    public String level2(@PathVariable("path") String path) {
        return "pages/level2/"+path;
    }

    /**
     * （Level3）绝世武林秘籍
     * @param path
     * @return
     */
    @GetMapping("/level3/{path}")
    public String level3(@PathVariable("path") String path){
        return "pages/level3/"+path;
    }

    /**
     * 使用 Shiro 编写认证操作
     */
    @PostMapping("/loginForm")
    public String loginForm(/*@RequestParam("username") String username,
                            @RequestParam("password") String password,*/
            User user, Model model){
        //  1、获取 Subject
        Subject currentUser = SecurityUtils.getSubject();
        //  2、封装用户数据
        UsernamePasswordToken token = new UsernamePasswordToken(user.getUsername(), user.getPassword());
        try {
            //  3、执行登录
            currentUser.login(token);
            //  记住我
            if(user.getRememberme() != null){
                token.setRememberMe(true);
            } else {
                token.setRememberMe(false);
            }
            return "redirect:/user/main";
        } catch (AuthenticationException e) {
            // 这里有一点需要注意：/login 若这个路径使用的是@GetMapping，那么将会报错，说
            // Resolved [org.springframework.web.HttpRequestMethodNotSupportedException: Request method 'POST' not supported]
            model.addAttribute("msg","账号或密码错误！");
            return "forward:/login";
        }
    }
}
```



## 3.4 编写 UserRealm

+ UserRealm.java

```java
package com.hzj.shiro.realm;

import org.apache.shiro.SecurityUtils;
import org.apache.shiro.authc.*;
import org.apache.shiro.authz.AuthorizationInfo;
import org.apache.shiro.authz.SimpleAuthorizationInfo;
import org.apache.shiro.realm.AuthorizingRealm;
import org.apache.shiro.subject.PrincipalCollection;
import org.apache.shiro.subject.Subject;

public class UserRealm extends AuthorizingRealm {
    /**
     * 执行授权逻辑
     * @param principalCollection
     * @return
     */
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
        Subject currentUser = SecurityUtils.getSubject();
        String principal = (String) currentUser.getPrincipal();
        SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();
        if("admin".equals(principal)) {
            info.addRole("level3");
        }
        if("manager".equals(principal)) {
            info.addRole("level2");
        }
        if("employee".equals(principal)) {
            info.addRole("level1");
        }
        return info;
    }

    /**
     * 执行认证逻辑
     * @param authenticationToken
     * @return
     * @throws AuthenticationException
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authenticationToken) throws AuthenticationException {
        UsernamePasswordToken token = (UsernamePasswordToken) authenticationToken;

//        System.out.println(token.getUsername());
//        System.out.println(token.getPassword());

        //  ！这里有一点需要注意：若判断 !"12345".equals(token.getPassword()，
        //  则会报错。至于为什么，我也不知道。
        if("admin".equals(token.getUsername()) || "manager".equals(token.getUsername()) || "employee".equals(token.getUsername())) {
            return new SimpleAuthenticationInfo(token.getUsername(),"12345","");
        } else {
            return null;
        }
    }
}
```



## 3.5 编写 ShiroConfig

+ ShiroConfig.java 

```java
package com.hzj.shiro.config;

import at.pollux.thymeleaf.shiro.dialect.ShiroDialect;
import com.hzj.shiro.realm.UserRealm;
import org.apache.shiro.spring.web.ShiroFilterFactoryBean;
import org.apache.shiro.web.mgt.DefaultWebSecurityManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.LinkedHashMap;
import java.util.Map;

@Configuration
public class ShiroConfig {

    /**
     * 创建 ShiroFilterFactoryBean
     * @param securityManager
     * @return
     */
    @Bean
    public ShiroFilterFactoryBean shiroFilterFactoryBean(DefaultWebSecurityManager securityManager) {
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
        //  设置安全管理器
        shiroFilterFactoryBean.setSecurityManager(securityManager);
        //  添加 shiro 内置过滤器
        Map<String,String> map = new LinkedHashMap<>();
        //  可以匿名访问 / 跳转到 welcome页面
        map.put("/","anon");
        //  可以匿名访问 /login 跳转到 登录页面
        map.put("/login","anon");
        //  可以匿名访问 /loginForm 提交表单信息
        map.put("/loginForm","anon");
        //  登出
        map.put("/logout","logout");

        //  role level1 可以访问 level1
        map.put("/level1/**","authc,roles[level1]");
        //  role level2 可以访问 level1、level2
        map.put("/level2/**","authc,roles[level2]");
        //  role level3 可以访问 level1、level2、level3
        map.put("/level3/**","authc,roles[level3]");


        //  其他页面都需要经过认证才能访问
        map.put("/**","authc");
        shiroFilterFactoryBean.setLoginUrl("/");
        shiroFilterFactoryBean.setUnauthorizedUrl("/");
        shiroFilterFactoryBean.setFilterChainDefinitionMap(map);

        return shiroFilterFactoryBean;
    }

    /**
     * 创建 DefaultWebSecurityManager
     * @param userRealm
     * @return
     */
    @Bean
    public DefaultWebSecurityManager defaultWebSecurityManager(UserRealm userRealm) {
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        //  关联 Realm
        securityManager.setRealm(userRealm);
        return securityManager;
    }

    /**
     * 创建 UserRealm
     * @return
     */
    @Bean
    public UserRealm userRealm() {
        return new UserRealm();
    }

    /**
     * 配置 ShiroDialect，用于 thymeleaf 和 shiro 标签配合使用
     * @return
     */
    @Bean
    public ShiroDialect shiroDialect(){
        return new ShiroDialect();
    }
}
```



# 四、SpringBoot 与分布式

## 4.1 分布式应用

​	在分布式系统中，国内常用zookeeper+dubbo组合，而Spring Boot推荐使用全栈的Spring，Spring Boot+Spring Cloud。

详情查看 SpringBoot高级.pdf

## 4.2 Zookeeper 和 Dubbo

+ ZooKeeper
  + ZooKeeper 是一个分布式的，开放源码的分布式应用程序协调服务。它是一个为分布式应用提供一致性服务的软件，提供的功能包括：配置维护、域名服务、分布式同步、组服务等。
+ Dubbo
  + Dubbo是Alibaba开源的分布式服务框架，它最大的特点是按照分层的方式来架构，使用这种方式可以使各个层之间解耦合（或者最大限度地松耦合）。从服务模型的角度来看，Dubbo采用的是一种非常简单的模型，要么是提供方提供服务，要么是消费方消费服务，所以基于这一点可以抽象出服务提供方（Provider）和服务消费方（Consumer）两个角色。

### 4.2.1 安装 zookeeper

```shell
:~# docker search zookeeper
:~# docker pull zookeeper
```

### 4.2.2 运行 zookeeper

```shell
:~# docker run --name zk01 -p 2181:2181 --restart always -d IMAGE ID
```

This image includes **EXPOSE** 2181 2888 3888 (the zookeeper **client port**，**follower port**，**election port** respectively). Since the Zookeeper "fails fast" it's better to always restart it.

### 4.2.3 入门案例（买卖门票）

1. <font color="#ff11ff">先创建一个Module：springboot-ticket，编写 TicketService 、TicketServiceImpl。</font>

+ TicketService.java

```java
package com.hzj.ticket.service;

public interface TicketService {
    public String getTicket();
}
```



2. <font color="#ff11ff">再创建一个Module：springboot-consumer，编写 UserService。</font>

   但是，在 UserService 中，需要用到 TicketService，因此，

   ​	**1）使用 dubbo，将 服务提供者 注册到 注册中心；**

   ​		小总结：

   ​		1.1）引入 dubbo 和 zookeeperClient 相关依赖

   ​		1.2）配置 dubbo 的扫描包和注册中心地址

   ​		1.3）使用 @Service 发布服务（dubbo的@Service）

   ```xml
   <!-- springboot-ticket、springboot-consumer 这两个Module都需要引入这两个依赖！ -->
   		<dependency>
               <groupId>org.apache.dubbo</groupId>
               <artifactId>dubbo</artifactId>
               <version>2.7.5</version>
        </dependency>
           <!-- 由于 dubbo 要操作 zookeeper，因此需要引入 zookeeper 客户端工具！ -->
           <dependency>
               <groupId>com.github.sgroschupf</groupId>
               <artifactId>zkclient</artifactId>
               <version>0.1</version>
           </dependency>
   ```

   + **springboot-10-dubbo\src\main\resources\application.properties**

   ```properties
   # ticket提供者
                        #       项目名称
   dubbo.application.name=springboot-10-dubbo
   dubbo.registry.address=zookeeper://120.25.164.172:2181
                       #将哪个包下的服务发出去
   dubbo.scan.base-packages=com.hzj.ticket.service
   
   #	写好配置之后，如何把配置发布出去呢？
   #在需要发布出去的类上加上注解 @org.apache.dubbo.config.annotation.Service; 注意是 dubbo的！
   ```

   + TicketServiceImpl.java

   ```java
   package com.hzj.ticket.service.impl;
   
   import com.hzj.ticket.service.TicketService;
   import org.apache.dubbo.config.annotation.Service;
   import org.springframework.stereotype.Component;
   
   @Component
   @Service    //将服务发布出去
   public class TicketServiceImpl implements TicketService {
       @Override
       public String getTicket() {
           return "《你的答案》";
       }
   }
   ```

   + Springboot10DubboApplication.java

   ```java
   package com.hzj;
   
   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.autoconfigure.SpringBootApplication;
   
   /**
    * 1、将服务提供者注册到注册中心
    *      1）引入 dubbo 和 zookeeperClient 相关依赖
    *      2）配置 dubbo 的扫描包和注册中心地址
    *      3）使用 @Service 发布服务
    */
   @SpringBootApplication
   public class Springboot10DubboApplication {
   
       public static void main(String[] args) {
           SpringApplication.run(Springboot10DubboApplication.class, args);
       }
   }
   ```

<img src="images\9.png" style="zoom:60%;" />



​			**2）使用 dubbo，将 服务提供者 注册到 注册中心；**

​				小总结：

​				2.1）引入 dubbo 和 zookeeperClient 相关依赖

​				2.2）配置 dubbo 的扫描包和注册中心地址

​				2.3）引用服务

+ **springboot-10-consumer\src\main\resources\application.properties**

```properties
dubbo.application.name=springboot-10-consumer
dubbo.registry.address=zookeeper://120.25.164.172:2181
```

+ UserService.java 里面要用到 TicketService，而且要**全类名相同**！因此将整个包复制过去，实现类可以删掉，只要 TicketService 接口即可。

<img src="images\11.png" style="zoom:60%;" />



+ 测试：
  + 测试的时候，调用接口的方法，接口的方法会远程调用实现类。这里两个 Module 都要启动。**但是**这里运行出现了 空指针异常。java.lang.NullPointerException。未解决。



# 五、SpringBoot 和 SpringCloud

 没学过



# 六、SpringBoot 热部署

## 6.1 引入依赖

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>
```

## 6.2 Ctrl + F9 重新编译即可

![](images\10.png)



# 七、SpringBoot 与监控管理

![](images\12.png)

## 7.1 引入依赖

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```

## 7.2 修改配置

+ 在1.5.x版本中通过management.security.enabled=false来暴露所有端点；现在2.x版本已经没用了。

+ 有以下方法可以实现：

  + 方式一：

  ```properties
  # 启用端点 env
  management.endpoint.env.enabled=true
   
  # 暴露端点 env 配置多个,隔开
  management.endpoints.web.exposure.include=env
  ```

  + 方式二：

  ```properties
  #	直接开启和暴露所有端点
  management.endpoints.web.exposure.include=*
  ```

+ **注意在使用Http访问端点时，需要加上默认/actuator 前缀**

## 7.3 启动主启动类

访问地址：http://localhost:8080/actuator；必须带上 **actuator**。

![](images\13.png)

+ 各个路径的作用，详情查看 另一个 pdf 文档。
