将阿里提供的[seata例子](https://github.com/alibaba/spring-cloud-alibaba/tree/master/spring-cloud-alibaba-examples/seata-example),修改为支持Docker, 并可以一键启动



### 快速体验

0. 安装docker, docker-compose



1. 启动服务

    修改配置, 是否模拟回滚

    * 正常事务, 没有异常抛出

       order-service/src/main/resources/application.properties, 修改**test.mockException=false**

    * 一定概率抛出异常,模拟回滚

       order-service/src/main/resources/application.properties, 修改**test.mockException=true**

    ```
    # 一键启动
    ./install.sh
    ```

    ​		

2. 验证:

   ```shell
   通过浏览器访问下面2个地址
   http://127.0.0.1:18081/seata/feign
   
   http://127.0.0.1:18081/seata/rest
   ```






### 增强点
本repo是依赖于 spring cloud alibaba的官方[示例](https://github.com/alibaba/spring-cloud-alibaba/tree/master/spring-cloud-alibaba-examples/seata-example), 使用过程中发现启动不是很方便, 所以修改为一键启动.
具体的增强点, 如下:

#### 创建网络

docker network create sc-net



#### 把各个Service封装到Docker


* IP替换为 docker容器名

  之前都是直接访问127.0.0.1的地址, 这样当封装到容器后, 每个服务都是跨容器访问其他容器提供的服务, 不在能通过内部ip访问了. 
  找到代码中的IP地址, 做类似于下面的替换


![image-20200906134538843](https://tva1.sinaimg.cn/large/007S8ZIlly1gigvqhz7o8j327w0ben0e.jpg)



* 添加docker插件

需要在**每个service**的pom.xml添加如下插件

```xml
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>1.0.0</version>
                <configuration>
                    <!-- 注意imageName一定要是符合正则[a-z0-9-_.]的，否则构建不会成功 -->
                    <imageName>cike/${project.artifactId}</imageName>
                    <dockerDirectory>${project.basedir}/src/main/docker</dockerDirectory>
                    <rm>true</rm>
                    <resources>
                        <resource>
                            <targetPath>/</targetPath>
                            <directory>${project.build.directory}</directory>
                            <include>${project.build.finalName}.jar</include>
                        </resource>
                    </resources>
                </configuration>
            </plugin>
```



* 编辑docker-compose文件

  * 基础中间件 docker-compose

  docker-compose/docker-compose.yml

  ```
  version: '3'
  services:
  
    nacos:
      image: nacos/nacos-server:1.1.3
      container_name: sc-nacos-standalone
      networks:
        - sc-net
      environment:
        - PREFER_HOST_MODE=hostname
        - MODE=standalone
      volumes:
        - ../data/nacos-server/logs/:/home/nacos/logs
      networks:
        - sc-net
      ports:
        - "8848:8848"
  
  
    redis: 
      image: redis:alpine
      container_name: sc-redis
      #restart: always
      volumes:
        - ../data/redis:/data
      environment:
        - REDIS_PASSWORD=123456
      networks:
        - sc-net
      ports:
        - 6379:6379
      env_file: .env
  
    rabbitmq:
      image: rabbitmq:management-alpine
      container_name: sc-rabbitmq
      #restart: always
      volumes:
        - ../data/rabbitmq:/var/lib/rabbitmq/mnesia
      networks:
        - sc-net
      ports:
        - 5672:5672
        - 15672:15672
      env_file: .env
  
    mysql:
      image: mysql:5.7
      container_name: sc-mysql
      #restart: always
      networks:
        - sc-net
      ports:
        - 3306:3306
      volumes:
        - ../data/mysql:/var/lib/mysql
      environment:
        TZ: Asia/Shanghai
        MYSQL_ROOT_PASSWORD: root123
  
    mysql-init:
      image: mysql:5.7
      command: /init-db.sh
      networks:
        - sc-net
      volumes:
        - ../db.init:/sql/db.init
        - ./init-db.sh:/init-db.sh
      environment:
        MYSQL_ROOT_PASSWORD: root123
  
  
    seata-server:
      image: seataio/seata-server:latest
      hostname: seata-server
      networks:
        - sc-net
      ports:
        - 8091:8091
      environment:
        - SEATA_PORT=8091
  
  networks:
    sc-net:
      external: true
  
  ```

  

  * 业务服务

  docker-compose/docker-compose.seata.sample.yml

  ```
  version: '3'
  services:
  
    storage-service:
      image: cike/storage-service
      container_name: storage-service
      #restart: always
      networks:
        - sc-net
      ports:
        - 18082:18082
      env_file: .env
      environment:
        TZ: Asia/Shanghai
  
    #授权服务
    account-service:
      image: cike/account-service
      container_name: account-service
      #restart: always
      networks:
        - sc-net
      ports:
        - 18084:18084
      env_file: .env
      environment:
        TZ: Asia/Shanghai
      depends_on:
        - storage-service
  
    business-service:
      image: cike/business-service
      container_name: business-service
      #restart: always
      networks:
        - sc-net
      ports:
        - 18081:18081
      env_file: .env
      environment:
        TZ: Asia/Shanghai
      depends_on:
       - order-service
  
      
    order-service:
      image: cike/order-service
      container_name: order-service
      #restart: always
      networks:
        - sc-net
      ports:
        - 18083:18083
      env_file: .env
      environment:
        TZ: Asia/Shanghai
      depends_on:
       - storage-service
  
  
  ```

  


* TODO:  通服务注册来声明服务

  不在通过容器名来访问服务, 这样不够灵活



### 单独调试其中一个业务服务

比如:order-service


``` shell
docker stop order-service
cd order-service
rm -rf target
mvn package && mvn docker:build
cd ../docker-compose 
docker-compose -f docker-compose.yml -f docker-compose.seata.sample.yml up -d order-service
cd ..
docker logs -f order-service
```

 

 