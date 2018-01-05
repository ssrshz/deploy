### deploy

#### Shell命令
1. 字符串替换 
```
domain='hello wor"ld'
domain1=${domain//\"/}
echo $(domain1)
hello world
```
2. shell 特殊符号用途

* $() 常用于将命令执行结果赋值给一个变量
* ${}使用变量
* 特殊输入用法,将标识之间的内容，可以包含变量，保存到.tar_env.env文件

```
cat>.tar_env.env<<tar_EOF
tar_env=${x}
tar_EOF
```
3. ansible 特殊用法

* serial 可以控制playbook一次更新多少台机器

```
- name: test play
  hosts: webservers
  serial: 3

- name: test play
  hosts: webservers
  serial: "30%"
  
- name: test play
  hosts: webservers
  serial:
  - 1
  - 5
  - 10
  
  - name: test play
  hosts: webservers
  serial:
  - "10%"
  - "20%"
  - "100%"
  
  - name: test play
  hosts: webservers
  serial:
  - 1
  - 5
  - "20%"
  
  - hosts: webservers
  max_fail_percentage: 30
  serial: 10
  
  ```
  
 * ansible --private-key参数
 
 
 指定ansible管理机器使用的远程控制的用户私钥，其对应公钥分发到被控制机器的authorized_keys，
 不依赖用户在管理端是否存在，便于后期自定义轮换密钥
 
 * handlers
 
 只有在所有的task都执行后(如果某个task执行失败 并且没有设置ignore_errors 后续的操作都不会再执行 包括被通告的handlers) handler才运行 而且只会运行一次 即使被多次被通告 handler按照在playbook中的先后顺序执行 而不是被通告的顺序
 
 主要用于善后的场景，如启动，重启，清理等
