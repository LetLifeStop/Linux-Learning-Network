### shell

unix上有许多解析器

1. sh
2. csh
3. ksh
4. tcsh
5. bash
6. shell

编写shell脚本：

把一堆命令组织到文件中，可以一次性执行所有的命令

```shell
#! /bin/bash     // 并不是注释，作用是指定解析器 
           
# this is to show what a examle looks like
           
echo "Our first example"
           
echo #this inserts an empty line in output
           
echo "we are currently in the following diractory"
          
echo "" 
          
/bin/pwd
          
echo    
          
echo "this direactory contaons"
          
/bin/ls                                                                     
```

当不指定解析器的时候，执行默认的解析器

#### 执行脚本方法：

1. ./sample.sh  ，前提是赋予执行权限
2. . sample.sh
3. /bin/sh sample.sh
4. source sample.sh 



#### 内建命令 

不需要bash解析器 ，也能执行

```shell
man bash-builtins
```



**关于小括号的应用**

```shell
cd ..;ls -l // 进入上一级命令，执行ls -l
```

```shell
(cd ..;ls -l) // 类比父子进程，父进程的环境没有变，但是子进程去进入到上级命令执行了ls -l 命令
```

​	

#### shell中的基本语法

**基本的数据类型**

string 类型

**基本的数据变量**

1. 环境变量（有点全局变量的感觉）
2. 本地变量

```shell
VAR=hello  // 创建一个本地变量,注意等号两边不要有空格
echo VAR   //  输出VAR
echo $VAR  //  输出hello
export VAR // 将VAR变量变成环境变量
unset VAR  // 删除对应的内容
```

```shell
alias pg='ps aux | grep'
```

自己创建一个组合命令



#### 文件名代换

\*   匹配多个字符

？匹配一个任意字符

[若干字符] 匹配方括号中任意一个字符的一次出现，如果多次出现的话，需要多个[]

**ex：**

如果要找t1234.sh的话,可以通过 ls t[0-4\][0-4\][2-5\][3-9\].sh 命令，还可以使用正则表达式（待填）



#### 命令代换

```shell
VAR=date
VAR=`date`   // 自动执行date命令
VAR=$(date)  // 自动执行date命令 
```



#### 算数代换

```shell
要计算40 + 33 是多少
方法：
VAR=40
方法1：echo $(($VAR+33))
方法2：ehco $((VAR+33))
方法3：echo $[VAR+33]
方法4：echo $[$VAR+33]
```

```shell
echo $[2#10+11] // 代表2进制下，10代表2，然后 2+11 = 13
```



#### 转义字符

用于去除紧跟其后的单个字符的特殊意义

```shell
ls \$SHELL
输出： $SHELL
如果要创建一个名称为 $ $test.sh的文件
方法： touch \$\ \$test.sh
如果要touch 创建一个名称为 -abc的文件,(-a 可能会被看成命令)
方法1： touch ./-abc
方法2： touch -- -abc 
```



#### 单引号与双引号

对于一个字符串，这两个是没有区别的

```shell
printf 'hello shell %d\n'  7
printf "hello shell %d\n"  7
```

这两个是没有区别的,但是

如果要输出 hello "xiaoming"

```shell
echo 'hello "xiaoming"'  
```



对于被双引号扩住的内容，将被视为单一字符。防止通配符扩展，但是允许变量扩展。

```shell
DATE=$(date)
echo "$DATE"  // 输出此时的时间
echo '$DATE'  // 输出$DATE
```

在shell中，如果要将当前的变量变成空

```shell
VAR=
```



#### shell脚本语法

```shell
echo $? 查看上一进程退出的值
```

**test命令**

```
test $var -gt 100 通过gt方式比较var 和 100的大小关系
echo $?  查看比较的结果
```

```shell
op 分类，-eq（等于） -ne（不等于）  -lt（小于） -le（小于等于） -gt（大于） -ge（大于等于）
```

**注意返回值，当为真的时候返回0，假的时候返回1**

![1566182982660](C:\Users\acm506\Desktop\Linux\Linux-Learning-Network\类型测试)

![1566183190880](C:\Users\acm506\Desktop\Linux\Linux-Learning-Network\测试)

**注意，[]也是一个命令，注意加空格**

```shell
var1=hello
var2=hello       // 等号两边不能有空格
[ $var1 = $var2 ]  // 等号两边必须有空格
```

逻辑运算

```shell
-a    逻辑与
-o    逻辑或
```

ex:

```shell
VAR=abc
[ -d Desktop -a "$VAR" = 'abc' ]  // 桌面为目录并且 VAR='abc'
echo $?
```



#### 作为一种好的shell编程习惯，应该把变量取值放在双引号中



### 分支语句



```shell
if [ -f ~/.bashrc ]; then   // 注意点，if后面和括号之间要有空格
   .~/.bashrc
elif [ -d ~/.bashrc ]; then 
   echo "hahhaha"
else
   printf "12231\ n"
fi   


if :    // 这个命令永远为真，类似于 if(1) 这种感觉
```

```shell
read str     // 这个的作用是输入str
```



**case/ esac 命令**

case命令类比C语言的switch /case 语句 ，每个匹配分块可以有若干命令，末尾必须以  ;;  结束



![1566199592731](C:\Users\acm506\Desktop\Linux\Linux-Learning-Network\case)

解释：

case 为判断分支  ，然后下面的yes|y|Yes|YES)  和 *）是判断条件  。 ;; 是类似于break的感觉。

然后以esac结尾 。**注意每一个判断的分支最后面的 ;; 是加在这个分支的最后面的**



#### 循环

**for循环**

```shell
for i in set; do 
   echo "I like $i"
done 
```

![1566201650181](C:\Users\acm506\Desktop\Linux\Linux-Learning-Network\for)

**测试代码如上**



**while循环**



![1566202144536](C:\Users\acm506\Desktop\Linux\Linux-Learning-Network\while)

限制3次输入

![1566202488695](C:\Users\acm506\Desktop\Linux\Linux-Learning-Network\while limit 3)

**break 和 continue**

break[n] 指定跳出几次循环 ，continue 一样



#### 位置参数和特殊变量

 

$0          相当于C语言main函数的argv[0]

$1、$2...   这些称为位置参数（Positional Parameter），相当于C语言main函数的argv[1]、argv[2]...

$#          相当于C语言main函数的argc - 1，注意这里的#后面不表示注释

$@          表示参数列表"$1" "$2" ...，例如可以用在for循环中的in后面。

$*          表示参数列表"$1" "$2" ...，同上

$?          上一条命令的Exit Status

$$          当前进程号

shift       从参数列表左移一个 ，$1 , $2 也依次往后类推（$0 也不算在其中 ）



#### 输入输出

echo 

```shell
echo "hello\n"    输出  hello\n
echo -e "hello\n" 输出  hello ,然后换行
```

tee命令

从标准输入读取，将读取到的内容写到标准输出 和 指定的文件中



**文件重定向**

1. cmd > file     把标准输出重定向到新文件中

2. cmd >> file  追加，将内容追加写到file文件的尾部

3. cmd > file 2>&1 标准出错也重定向到所制定的文件（把标准输出重定向到file，然后把标准出错重定向到标准输出，所以就实现了标准出错重定向到所指定的文件）
4. cmd < file1 > file2  从file1 中读取，然后写到file2 中
5. cmd < &fd  把文件描述符fd作为标准输入
6. cmd > &fd  把文件描述符fd作为标准输出
7. cmd < &-     关闭标准输入



#### 函数

shell也有函数的概念，但是函数概念没有返回值也没有参数列表

![1566205452151](C:\Users\acm506\Desktop\Linux\Linux-Learning-Network\func)

函数传参的方法：

![1566205912129](C:\Users\acm506\Desktop\Linux\Linux-Learning-Network\函数传参的方法)

结果：

```shell
.func.sh 44 55
bash 
44
55
start
bash
11
22
dadwdaw
--end--
```

注意 $0 都是表示当前进程名



练习：

循环创建多个目录

```shell
  1 #!/bin/bash
  2       
  3 is_directory(){
  4     DIR_NAME=$1
  5     if [ ! -d $DIR_NAME ]; then
  6     ┊   return 1
  7     else
  8     ┊   return 0
  9     fi
 10 }     
 11       
 12 for DIR in "$@"; do
 13     if is_directory "$DIR"
 14     then :
 15     else
 16     ┊ echo "$DIR dosen't exit, creating now---"
 17     ┊ mkdir $DIR > /dev/null 2>&1
 18     ┊ if [ $? -ne 0 ]; then                                                 
 19     ┊   ┊ echo "create director error"
 20     ┊   ┊ exit 1
 21     ┊ fi
        fi
   done     
```



#### shell 脚本调试方法

-n 读一遍脚本中的命令但是不执行，用于检查脚本中的语法错误

-v 一边执行脚本，一边讲执行过的脚本命令打印到标准错误输出

-x  提供跟踪执行信息，将执行的每一条命令和结果依次打印出来

使用方法：

1. 在命令行提供参数。

```shell
$ sh -x ./script.sh
```

1. 在脚本开头提供参数

        ```shell
#! /bin/sh -x
        ```

3. 在脚本中用set命令限制范围 , -x 是调试起点，+x是调试终点

```shell
set -x
1111
1111
1111
set +x 
```



