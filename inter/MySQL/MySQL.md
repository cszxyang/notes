### 查询优化

- 对频繁查询列建立索引
- 将唯一索引列设置为非空，避免索引失效

#### SQL注入

由于传入的参数与系统的SQL拼接成了合法的SQL而导致的，而其本质还是将用户输入的数据当做了代码执行。

- 对输入做格式限制，如登录时的name或者password不允许带有%之类的符号
- 使用网络应用防火墙（WAF），过滤异常流量
- 尽量使用预编译SQL语句

#### 安全部署

强密码、最小特权原则

### 主从复制



### 安全