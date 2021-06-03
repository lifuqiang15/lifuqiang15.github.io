# ES集群x-pack配置



### 啥是x-pack？

```
X-Pack是Elastic Stack扩展功能，提供安全性，警报，监视，报告，机器学习和许多其他功能。 ES7.0+之后，默认情况下，当安装Elasticsearch时，会安装X-Pack，无需单独再安装。

自6.8以及7.1+版本之后，基础级安全永久免费
```



##### 为 ES 集群创建节点认证中心

```
./bin/elasticsearch-certutil  ca

可以设置一个密码，也可以直接回车。
默认文件会在 ES 根目录产生，名为 elastic-stack-ca.p12。
然后可以将文件 elastic-stack-ca.p12 复制到每个 ES 节点的根目录下
```



##### 为集群中的每个节点创建证书和私钥（每个节点都要执行以下内容）

生成证书和密钥。

```
./bin/elasticsearch-certutil cert --ca ./elastic-stack-ca.p12
可以设置密码，也可以直接回车。
默认会生成文件 elastic-certificates.p12
```



将两个证书文件elastic-stack-ca.p12  elastic-certificates.p12 移动到config下

```
mv elastic-stack-ca.p12  elastic-certificates.p12 config
```



##### 修改ES配置文件

```
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path:   /data/es/elasticsearch-7.8.0/config/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: /data/es/elasticsearch-7.8.0/config/elastic-certificates.p12
```

##### 如果之前节点证书设置了密码，将密码添加到 keystore

```
./bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
./bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password
```



##### 重启ES集群

设置内置用户密码

```
./bin/elasticsearch-setup-passwords interactive 


核心：
auto - 随机生成密码。
interactive - 自定义不同用户的密码。

注意：必须配置好xpack之后，才能设置密码,则会报错。
```



外置账号

```
 ./elasticsearch-users useradd elastic -p XXXXX -r superuser 
```

修改 Kibana 配置文件，访问 ES

```
elasticsearch.username: "elastic"
elasticsearch.password: "elastic"
xpack.security.enabled: true
```



浏览器访问

![image-20210528184328967](assets/image-20210528184328967.png)



表示密码配置成功



##### 使用curl访问

```
curl  -u elastic:123123  http://192.168.30.201:9200/_cat/indices
```





# kibana的权限控制



打开StackManagement，看到Security 列表里有Users和Roles两项

![image-20210531154953940](assets/image-20210531154953940.png)



### 创建角色

点击Roles

![image-20210531155126233](assets/image-20210531155126233.png)



点击Createrole

![image-20210531155449894](assets/image-20210531155449894.png)



### 创建用户

角色创建完后，创建用户

回到security菜单，点击user

![image-20210531155532778](assets/image-20210531155532778.png)



点击create user

![image-20210531155603570](assets/image-20210531155603570.png)



# 权限管理的API



### 查看所有用户

```
elastic  123123 分别是超级用户的用户名和密码
curl -X GET -u elastic:123123 "localhost:9200/_xpack/security/user"
```



### 查看指定用户，例如张三	

```
elastic  123123 分别是超级用户的用户名和密码
curl -X GET -u elastic:123123  "192.168.30.201:9200/_xpack/security/user/zhangsan"
```



### 创建用户

```
elastic  123123 分别是超级用户的用户名和密码
例如添加用户jacknich：

curl -X POST -u elastic:123123  "localhost:9200/_xpack/security/user/jacknich" -H 'Content-Type: application/json' -d'
{
  "password" : "j@rV1s",  用户名密码，必须
  "roles" : [ "admin", "other_role1" ], 角色，必须
  "full_name" : "Jack Nicholson",
  "email" : "jacknich@example.com",
  "metadata" : {
    "intelligence" : 7
  }
}
```



### 修改密码

```
curl -X POST "localhost:9200/_xpack/security/user/jacknich/_password" -H 'Content-Type: application/json' -d'
{
  "password" : "s3cr3t"
}
```



### 禁用/启用/删除用户

```
这里以jacknich为例
curl -X PUT "localhost:9200/_xpack/security/user/jacknich/_disable"
curl -X PUT "localhost:9200/_xpack/security/user/jacknich/_enable"
curl -X DELETE "localhost:9200/_xpack/security/user/jacknich"
```



### 角色管理API

```
curl -X GET "localhost:9200/_xpack/security/role"
curl -X GET "localhost:9200/_xpack/security/role/my_admin_role"
curl -X DELETE "localhost:9200/_xpack/security/role/my_admin_role"
curl -X POST "localhost:9200/_xpack/security/role/my_admin_role" -H 'Content-Type: application/json' -d'
{
  "cluster": ["all"],
  "indices": [
    {
      "names": [ "index1", "index2" ],
      "privileges": ["all"],
      "field_security" : { // 可选
        "grant" : [ "title", "body" ]
      },
      "query": "{\"match\": {\"title\": \"foo\"}}" // 可选
    }
  ],
  "run_as": [ "other_user" ], // 可选
  "metadata" : { // 可选
    "version" : 1
  }
}
'
```



### 基于角色的访问控制（RBAC）

X-Pack安全性提供了一种基于角色的访问控制(RBAC)机制，它使你能够通过向角色分配特权和向用户或组分配角色来授权用户。







# kibana基于nginx做"免密"登录



kibana的dashboard功能非常丰富，在一些场景下我们需要直接将其分享的仪表盘以iframe方式嵌入到业务系统中，但业务系统已经登录了一次了，再嵌入的dashboard又需要登录一次（es配置了x-pack），用户体验不太好，因此需要进行“免密”代理

这里提供一个nginx代理授权header头的方式，配置如下：



### Kibana的配置

```
server.port: 5601
server.host: "0.0.0.0"    
server.basePath: "/kibana"    # 这里的path要和nginx中location一致
server.rewriteBasePath: true  # 这一行要配置，否则访问的时候会提示重定向过多

elasticsearch.requestHeadersWhitelist: [authorization]
elasticsearch.customHeaders: {}
server.name: "192.168.30.203"
elasticsearch.hosts: ["http://192.168.30.201:9200"]
elasticsearch.username: "elastic"  // 登录es的用户名，elastic相当于root用户
elasticsearch.password: "123123"   // 密码
```



### nginx配置

```
  location  ^~  /kibana {
             proxy_pass   http://192.168.30.203:5601/kibana;
             proxy_set_header  Host $host;
             proxy_set_header  X-Real-IP $remote_addr;
             proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
             proxy_set_header  Authorization "Basic base64(username:password);
             #rewrite ^/kibana/(.*)$ /$1 break;
        }

注意：username为登录的用户，password为登录的密码
其中的base64(username:password)是   用户名:密码  的base64格式，可在系统下直接输入命令获得：

echo -n elastic:123123 | base64
ZWxhc3RpYzoxMjMxMjM=

然后proxy_set_header  Authorization "Basic ZWxhc3RpYzoxMjMxMjM="; 这一行这么配置就可以了

重启nginx后就可直接内嵌到业务系统了

```









































































































































































