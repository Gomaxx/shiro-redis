#分支描述
在shiro的一次认证过程中会调用10次左右的doReadSession，如果使用内存缓存这个问题不大。但是如果使用redis，而且子网络情况不是特别好的情况下这就成为问题了。我简单在我的环境下测试了一下。一次redis请求需要80 ~ 100 ms， 一下来10次，我们一次认证就需要10 * 100 = 1000 ms, 这个就是我们无法接受的了。
针对这个问题可以重写DefaultWebSessionManager，将缓存数据存放到request中，这样可以保证每次请求（可能会多次调用doReadSession方法）只请求一次redis。

# shiro-redis

[![Build Status](https://travis-ci.org/alexxiyang/shiro-redis.svg?branch=master)](https://travis-ci.org/alexxiyang/shiro-redis)


shiro only provide the support of ehcache and concurrentHashMap. Here is an implement of redis cache can be used by shiro. Hope it will help you!

How to use it?
===========

You can choose these 2 ways to include shiro-redis into your project
* use "git clone https://github.com/alexxiyang/shiro-redis.git" to clone project to your local workspace and build jar file by your self
* add maven dependency 

```xml
    <dependency>
  		<groupId>org.crazycake</groupId>
  		<artifactId>shiro-redis</artifactId>
  		<version>2.4.2.1-RELEASE</version>
  	</dependency>
```

How to configure ?
===========
You can choose 2 ways : shiro.ini or spring-*.xml

shiro.ini:

```properties
#redisManager
redisManager = org.crazycake.shiro.RedisManager
#optional if you don't specify host the default value is 127.0.0.1
redisManager.host = 127.0.0.1
#optional , default value: 6379
redisManager.port = 6379
#optional, default value:0 .The expire time is in second
redisManager.expire = 30
#optional, timeout for jedis try to connect to redis server(In milliseconds), not equals to expire time! 
redisManager.timeout = 0
#optional, password for redis server
redisManager.password = 

#============redisSessionDAO=============
redisSessionDAO = org.crazycake.shiro.RedisSessionDAO
redisSessionDAO.redisManager = $redisManager

# modify by goma 
#sessionManager = org.apache.shiro.web.session.mgt.DefaultWebSessionManager
sessionManager = org.crazycake.shiro.MyDefaultWebSessionManager

sessionManager.sessionDAO = $redisSessionDAO
securityManager.sessionManager = $sessionManager

#============redisCacheManager===========
cacheManager = org.crazycake.shiro.RedisCacheManager
cacheManager.redisManager = $redisManager
#custom your redis key prefix, if you doesn't define this parameter shiro-redis will use 'shiro_redis_session:' as default prefix
cacheManager.keyPrefix = users:security:authz:
securityManager.cacheManager = $cacheManager
```

spring.xml:
```xml
<!-- shiro filter -->
<bean id="ShiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
	<property name="securityManager" ref="securityManager"/>
	
	<!--
	<property name="loginUrl" value="/login.jsp"/>
	<property name="successUrl" value="/home.jsp"/>  
	<property name="unauthorizedUrl" value="/unauthorized.jsp"/>
	-->
	<!-- The 'filters' property is not necessary since any declared javax.servlet.Filter bean  -->
	<!-- defined will be automatically acquired and available via its beanName in chain        -->
	<!-- definitions, but you can perform instance overrides or name aliases here if you like: -->
	<!-- <property name="filters">
		<util:map>
			<entry key="anAlias" value-ref="someFilter"/>
		</util:map>
	</property> -->
	<property name="filterChainDefinitions">
		<value>
			/login.jsp = anon
			/user/** = anon
			/register/** = anon
			/unauthorized.jsp = anon
			/css/** = anon
			/js/** = anon
			
			/** = authc
		</value>
	</property>
</bean>

<!-- shiro securityManager -->
<bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager">

	<!-- Single realm app.  If you have multiple realms, use the 'realms' property instead. -->
	
	<!-- sessionManager -->
	<property name="sessionManager" ref="sessionManager" />
	
	<!-- cacheManager -->
	<property name="cacheManager" ref="cacheManager" />
	
	<!-- By default the servlet container sessions will be used.  Uncomment this line
		 to use shiro's native sessions (see the JavaDoc for more): -->
	<!-- <property name="sessionMode" value="native"/> -->
</bean>
<bean id="lifecycleBeanPostProcessor" class="org.apache.shiro.spring.LifecycleBeanPostProcessor"/>	

<!-- shiro redisManager -->
<bean id="redisManager" class="org.crazycake.shiro.RedisManager">
	<property name="host" value="127.0.0.1"/>
	<property name="port" value="6379"/>
	<property name="expire" value="1800"/>
	<!-- optional properties:
	<property name="timeout" value="10000"/>
	<property name="password" value="123456"/>
	-->
</bean>

<!-- redisSessionDAO -->
<bean id="redisSessionDAO" class="org.crazycake.shiro.RedisSessionDAO">
	<property name="redisManager" ref="redisManager" />
</bean>


<!-- modify by goma -->
<!-- sessionManager
<bean id="sessionManager" class="org.apache.shiro.web.session.mgt.DefaultWebSessionManager">
	<property name="sessionDAO" ref="redisSessionDAO" />
</bean>
-->
<bean id="sessionManager" class="org.crazycake.shiro.MyDefaultWebSessionManager">
	<property name="sessionDAO" ref="redisSessionDAO" />
</bean>


<!-- cacheManager -->
<bean id="cacheManager" class="org.crazycake.shiro.RedisCacheManager">
	<property name="redisManager" ref="redisManager" />
</bean>
```

> NOTE
> Shiro-redis don't support SimpleAuthenticationInfo created by this constructor `org.apache.shiro.authc.SimpleAuthenticationInfo.SimpleAuthenticationInfo(Object principal, Object hashedCredentials, ByteSource credentialsSalt, String realmName)`.
> Please use `org.apache.shiro.authc.SimpleAuthenticationInfo.SimpleAuthenticationInfo(Object principal, Object hashedCredentials, String realmName)` instead.

If you found any bugs
===========

Please send email to idante@qq.com
