#Crossrealm in Hadoop Authorized by Kerberos

##Step-by-Step

###Client Version
访问的客户端至少需JDK 1.7，原因是kerberos跨域时JDK bug仅在1.7以上修复。

###Add Principals
以A访问B为例，两个Realm均需要添加一个krbtgt/B@A的principal，其中两个principals的密码、加密方式和kv对需一致。

###krb5.conf of Client
Client端的krb5.conf需指定domain-realm的映射:

```
[domain_realm]
.a.hadoop = A.COM
.b.hadoop = B.COM
```  
添加这一配置的原因是Client需要确定自己需要访问的是哪一个realm。

###Hadoop Client
hadoop的hdfs-site.xml需添加以下配置：  

```
<property>
    <name>dfs.namenode.kerberos.principal.pattern</name>
    <value>*</value>
</property>
```
yarn-site.xml需添加以下配置：

```
<property>
    <name>yarn.resourcemanager.principal.pattern</name>
    <value>*</value>
</property>
```
配置该参数的目的在于获取remote server的principal，方法调用自  
hadoop-common: org.apache.hadoop.security.SaslRpcClient.getServerPrincipal()  
该方法将比较需访问的跨域principal和该方法生成的principal（或principal匹配规则）  
若不配置此pattern，那么该方法会根据Clietn所在realm生成一个principle。因为Client只可能属于一个principle，所以跨域的访问请求指定的principal不可能等于生成的principal，无法跨域。  
若配置此项，该方法会根据pattern翻译成正则表达式，若匹配则通过。  
例子中将hdfs和yarn层面的principal.pattern均设置成通配，则该客户端访问任一realm的请求在Client端都是可以通过的。

###Auth\_to\_Local
hadoop的core-site.xml配置需添加以下property：

```
<property>
    <name>hadoop.security.auth_to_local</name>
    <value>
      DEFAULT
      RULE:[1:$1](sync_.*)
      RULE:[2:$1](sync_.*)
    </value>
</property>
```
该参数用于server将principal映射成一个本地账号进行后续操作。该方法调用自：  
hadoop-auth: org.apache.hadoop.security.authentication.util.KerberosName.getShortName()    
如果不设该值，则取值为DEFAULT，server验证时将仅通过其所在realm的访问请求。  
RULE的配置分为三部分：

* 翻译principal  
[1:$1&$0]将匹配如service@REALM不包含hostname的principal并翻译成service&REALM。  
[2:$1&$2&$0]将匹配如service/hostname@REALM的principal并翻译成service&hostname&REALM
* filter  
filter的配置不是正则表达式，而是类似于shell的GlobalPattern，server会将Global Pattern翻译成正则进行匹配。若规则为DEFAULT，则会以service作为local账号。
* replace  
替换的规则为s/\*/\*/g，具体请直接参考shell中的sed命令。  

例子中为仅允许以sync\_开头的service，以principal全称（不替换）作为local账号的auth\_to\_local配置。

###PS
1. auth\_to\_local在NameNode和ResourceManager的core-site.xml中都需要配置，前者在hdfs层面访问文件，后者在yarn层面提交任务。
2. 添加rules配置时，不要使用如[1:$1&$2&$0]和[2:$1&$2&$3&$0]等类似配置，虽然这是一个错误配置，但是同样会触发一个hadoop尚未修复的bug。
  
