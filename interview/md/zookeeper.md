## Zookeeper

### 一般命令
create [-s] [-e] /path data 创建一个节点.
- -s为有序节点, 会自动在节点名字后面加1,2,3,4等序号, 
- -e为临时节点, 客户端断开后自动删除
    
ls /path 查看子节点

get /path 查看节点状态
- cZxid
- ctime
- mZxid
- mtime
- cversion
- dataVersion
- aclVersion
- dataLength
- numChildren
- ephemeralOwner

set path data [version] 设置值, 乐观锁

delete /path [version] 删除节点, 需要version匹配, 乐观锁

### watcher事件
> 当监控的某个对象(znode)发生变化, 会触发watcher事件

一次性, 触发后立即销毁, 设置一次最多触发一次

父节点,子节点 增删改都触发watcher
事件类型
1. 节点创建事件         NodeCreated
2. 节点删除事件         NodeDeleted
3. 节点数据变化事件      NodeDataChanged
4. 子节点创建事件        NodeChildrenChanged
5. 子节点删除事件        NodeChildrenChanged    
    
    
#### 设置watcher
get /path [watch] 返回路径详情并设置watcher

stat /path [watch]  可以为空目录创建事件, 目录创建时触发事件,节点有值才会触发事件



#### watcher使用场景
- 统一资源配置



### ACL access control lists 权限控制列表
> 针对节点设置 读 写 权限, 保障数据安全

[scheme:id:permissions] 构成权限列表

scheme      代表采用哪种类型权限机制
- world     全局, 下面只有一个用户, 就是anyone
- auth      认证登陆, 需要注册用户有权限 auth:user:password:[permissions]  明文密码, 很少用
- digest    需要口令才能访问   digest:username:BASE64(SHA1(password)):[permissions] 密文密码
- ip        指定ip才能访问
- super     超级管理员, 所有权限, 重启后有效 启动时添加 "-Dzookeeper.DigestAuthenticationProvider.superDigest=xl:PIQO454VsRUMlaFKnVub5uEszA0="环境参数

id          用户
- anyone  所有人

permissions 权限组合字符串
- c create 创建, 代表有创建子节点的权限
- r read 获取当前节点/子节点权限
- d delete 删除子节点的权限
- w write 写权限, 设置节点数据
- a admin 设置权限的权限

#### 权限示例
world:anyone:cdrwa 任何人有任何权限

digest:xl:xl:cdrwa xl的用户使用密码xl登陆有任何权限

#### acl命令 TODO: 下面三个命令不太明白
getAcl /path 显示权限, 永远显示第一个设置权限的用户

setAcl /path acl 设置权限
- setAcl /names/xl auth:xl:xl:crdwa
- setAcl /names/xl digest:xl:PIQO454VsRUMlaFKnVub5uEszA0=:crdwa
- setAcl /names/xl auth::crdwa      匿名设置, 设置第一个用户的权限

addauth  scheme auth  添加口令, 使得注册时需要口令, 同时也是登陆口令
- addauth digest xl:xl   注册一个使用digest登陆的用户xl, 密码为xl


#### acl使用场景
- 开发/测试环境分离, 开发无权操作测试库的节点,只能看
- 生产环境控制指定ip的服务可以访问相关节点, 当然动态ip就不行



### 四字命令 four letter words
https://zookeeper.apache.org/doc/r3.4.14/zookeeperAdmin.html#sc_zkCommands
-  echo stat | nc 127.0.0.1 2181   查看zookeeper状态
-  echo ruok | nc 127.0.0.1 2181   are you ok? 返回imok ....醉了
-  echo dump | nc 127.0.0.1 2181   查看session 会话
-  echo conf | nc 127.0.0.1 2181   查看服务器配置
-  echo cons | nc 127.0.0.1 2181   查看连接到服务器的刻客户端信息
-  echo envi | nc 127.0.0.1 2181   查看环境变量
-  echo mntr | nc 127.0.0.1 2181   查看健康信息
-  echo wchs | nc 127.0.0.1 2181   查看watcher数量
-  echo wchc | nc 127.0.0.1 2181   查看watcher与session, 需要设置白名单 whilelist
-  echo wchp | nc 127.0.0.1 2181   查看watcher与path, 需要设置白名单 whilelist
-  echo wchs | nc 127.0.0.1 2181   查看watcher信息


### 集群
> 主从节点, 心跳机制


