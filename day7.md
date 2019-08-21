#### 正则表达式

egrep  对文件中的正则表达式来匹配

#### 字符类

![1566284880377](C:\Users\acm506\AppData\Roaming\Typora\typora-user-images\1566284880377.png)

#### 数量限定符

![1566285808756](C:\Users\acm506\AppData\Roaming\Typora\typora-user-images\1566285808756.png)

**再次注意grep找的是包含某一模式的行，而不是完全匹配某一模式的行。**

#### 位置限定符

 ![1566284938936](C:\Users\acm506\AppData\Roaming\Typora\typora-user-images\1566284938936.png)                                                 

#### 其它特殊字符

![1566284954837](C:\Users\acm506\AppData\Roaming\Typora\typora-user-images\1566284954837.png)



**Basic正则 和 Extended 正则的区别**

如果是Extended 的话，需要对字符 ？ +  {} | （） 进行特殊转义



### sed

流编辑器 ，类似于vim编辑器中的末行模式

```shell
sed '/echo/s/echo/printf/g' 将包含echo的行，通过替换的功能，将echo替换成 printf ，如果出现多次，就多次替换
```

### grep:

格式及主要参数：

grep [options]

主要参数：  grep --help可查看

​    -c：只输出匹配行的计数。

​    -i：不区分大小写。

​    -h：查询多文件时不显示文件名。

​    -l：查询多文件时只输出包含匹配字符的文件名。

​    -n：显示匹配行及 行号。

​    -s：不显示不存在或无匹配文本的错误信息。

​    -v：显示不包含匹配文本的所有行。

​    --color=auto ：可以将找到的关键词部分加上颜色的显示。



sed 

**sed命令不会修改源文件，如果要修改记得加参数 -i；如果不加的话，只是显示修改过后的样子，但是源文件并没有修改**

基本格式

1.  sed 参数    ‘脚本语句’    待操作文件

  ```shell
sed -i ‘4a xxxxxxxxx' case.sh
  ```

2. sed  参数  -f  ‘脚本文件’ 待操作的文件

参数

a  追加

i  插入

d  删除

s  替换

```shell
sed 's/BUF/buffer/g' case.sh
sed 's/BUF/--&--/g' case.sh   这里&指代的是 BUF
（错误）sed 's/([0-9])([0-9])/-\1-~\2~/' 这里的 \1 代表第一个[0-9] , \2类似，这依据的作用就是将 第一个左右加上 - ，第二个左右加上 ~  
（正确）sed -i 's/\([0-9]\)\([0-9]\)/~\1~-\2-/' w  ，默认使用Extended正则
```

**常用sed命令**

```shell
/pattern/p  打印匹配pattern的行
/pattern/d  删除匹配pattern的行
```

如果是一次处理多个命令的话，两种方法如下：

```shell
sed -e 's/yes/no/' -e 's/yes/no/' w
sed 's/yes/no/;s/yes/no/' w
```



**正则表达式是按照贪心的模式来进行设计的**，但是对于如下情况，

```html
<html><head><title>hello world </title>
```

我们要将除了hello world 之外的东西都去掉，这样的话 <*>匹配的是整个串，这样弊端就漏出来了。

正确方法如下：

```shell
sed 's/<[^<>]>//g' t.html
```



####  awk

BEGIN： 



与sed相呼应，处理列数据

默认是按照空格进行拆分

```shell
ps aux | awk '{print $2}' //获取当前的所有进程,2代表第二列 ， 0代表都打印
```



```shell
awk -F: '{print $6}' /etc/passwd  // 强制按照 ： 来进行拆分 
```

格式：

```shell
awk option 'script' file1 file2 ,,,,
```

option:

BEGIN：pattern 未匹配文件之前，执行某些操作

awk 'BEGIN {FS=':'}{print $7}'  /etc/passwd 

END：pattern未匹配文件结束，执行某些操作

```shell
/pattern/{actions}
condition{action}
```

  统计文件中有多少空行

```shell
awk '/^ *$/ {x=x+1} END {print x;}' w
```

/^ *$/ 表示空格开头，然后n个空格，然后以空格结尾

输出方式，

1. printf("%d %d\n", $t1$,t2);

2. print $t1 $t2   (自带换行)



在AWK 和 SED 中，变量 和 shell 相比，不再使用 $ 符号，直接赋值可以使用 (awk 中的 $1 代表第一列)



#### C程序中使用正则

regcomp（），regexec（），regfree（），regerror（）

编译                    匹配                     释放

```c
int regcomp(regex_t *compiled ,const char *pattern , int cflags)
```

参数1：结构体， 存储正则表达式
参数2：正则表达式串

参数3：标志位：

​     REG_EXTENDED     扩展正则

​     REG_ICASE              忽略大小写

​     REG_NEWLINE       识别换行符

​    REG_NOSUB   不用存储匹配后的结果，只返回是否匹配成功。如果设置该标志位，那么在下面的函数将忽略nmatch参数和pmatch参数

返回值：成功0；失败错误号

```c
int regexec(regex_t *complied ,char *string , size_t nmatch ,regmatch_t matchptr[],int eflags)
```

```c
typedef struct {
regoff_t rm_so;
regoff_t rm_eo;
}regmatch_t;
```



参数1：regcomp 变异后传出的结构体

参数2 ：待用正则表达式进行匹配的字符串

参数3：数组的大小,内部都是regmatch_t 结构体

参数4：用来存储返回结果的数组

参数5： REG_NOTBOL  让特殊字符 ^ 无作用

​              REG_NOTEOL   让特殊字符 $ 无作用

```c
void regfree(regex_T *complied)
```

释放结构体

```c
size_t regerror(int errcode ,regex_t *compiled , char *buffer,size_t length)
```

处理错误

示例：

```c
/*************************************************************************
	> File Name: regex.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月21日 星期三 11时23分09秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<regex.h>
#include<sys/types.h>

int main(int argc ,char *argv[]){

    if(argc != 3){
        printf("tai shao le aaaaaa\n");
        return 1;
    }

    const char * p_regex_str = argv[1];
    const char * p_txt = argv[2];
    regex_t oregex;
    int ret = 0 ;
    char emsg[1024] = {0};
    size_t emsg_len = 0;

    if((ret = regcomp(&oregex , p_regex_str ,REG_EXTENDED|REG_NOSUB)) == 0 ){
        if((ret = regexec(&oregex , p_txt , 0 , NULL , 0)) == 0){
            printf("%s  matches %s\n" , p_txt , p_regex_str);
            regfree(&oregex);
            return 1;
        }
    }
  
    emsg_len = regerror(ret , &oregex , emsg , sizeof(emsg));
    emsg_len = emsg_len < sizeof(emsg) ? emsg_len : sizeof(emsg);

    emsg[emsg_len] = '\0';
    printf("regex error Msg : %s\n" , emsg);

    regfree(&oregex);
     
    return 1;
}
```



####  find补充

参数

1.    \- maxdepth   深度

  ```shell
find  ./ -maxdepth 1 -type d
目录只递归一次，输出目录的个数
find  ./ -maxdepth 1 -name "*.sh" -exec ls -lh {} \ ;
exec执行ls -lh ，然后{} 对应 前面的 name之前的内容，\代表对分号进行转义
find  ./ -maxdepth 1 -name "*.sh" -ok ls -lh {} \ ;
起到的作用和上面的相同
  ```

2. -xargs 可以和管道配合使用,在区分的时候通过空格，换行等区分

```shell
find  ./ -maxdepth 1 -name "*.sh" | xargs ls -l
find ./ -maxdepth 1 -type f -print0 | xargs -0 ls -lh
在每个输出的后面加上 -0 ，然后xargs 区分的时候通过-0 来区分
```

xargs 和 exec的区别：

exec 是 通过 对前面的结果集一次性投入到结果集 中 

xargs 是 对前面的结果来进行分批处理 

3. \- atime   访问时间
4. -mtime   文件内容修改时间
5. -ctime     文件属性修改时间

-mtime +5 5天以前

-mtime -5 5天以内

