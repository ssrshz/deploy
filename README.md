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
