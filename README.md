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

**$() 常用于将命令执行结果赋值给一个变量
