
VIM 命令
=====

#### 搜索

- `/` 搜索

```shell
/isTestSource="true"
```

直接输入/ + 字符串，然后按回车
之后按n跳转到下一个匹配
按N跳转到上一个匹配

`*`跳转到光标所在单词的下一个匹配
`#`跳转到光标所在单词的上一个匹配

- 查看所有匹配

```shell
:g/isTestSource="true"/#
:g/字符串/p  #显示所有匹配的行（无行号）
:g/字符串/=  # 显示行号但不显示内容
```

- 搜索并删除

```shell
:g/isTestSource="true"/d  
:g/字符串/  #仅匹配不执行操作
:g/字符串/dc  #删除没一行前询问
```

- 高亮

```shell
:match Search /.*字符串.*/    #高亮所有匹配的行
```

- 搜索并替换

```shell
:s/old/new/      #将当前行中第一个匹配的old替换为new
:s/old/new/g     #将当前行中所有匹配的old替换为new
:%s/old/new/g    #在整个文件范围内替换所有匹配的old为new
:5,10s/old/new/g  #在第5行到第10行范围内替换所有匹配
```