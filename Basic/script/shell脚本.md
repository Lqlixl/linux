# 知识点 
## 循环结构
选择结构：
### if:单分支·双分支·多分支
### 单分支：
```bash
if CONDITION;then
  statement
  ...
fi
#### 双分支：
if CONDITION;then
  statement
  ...
else
  statement
  ...
fi
#### 多分支:
if CONDITION;then
  statement
  ...
elif CONDITION;then
  statement
  ...
else
  statement
  ...
fi

#### case语句；选择结构
case SWITCH(是引用的变量要加 $ ) in
  value1)
      statement
      ...
      ;;
  value2)
    statement
    ...
    ;;
  *)
    statement
    ...
    ;;
esac
```

### for循环  ,缺点，循环由列表决定，没列表就不行。像真假for循环就不行。
```bash 
for 变量 in 列表；do
    循环体
done
```
#### c语言风格的:
```bash
for (( exp1; exp2; exp3));do
  COMMANDS;
  done

执行  exp1  ，然后 执行  exp2  ，如果 2 flase 就 exit ，如果 exp2 ture, 就执行 循环体 { COMMANDS exp3 组成的循环体} ,其中exp3是循环体的最后一条命令。

#1

```

### while循环 , 可以代替 for 循环
```bash
while CONDITION;do    \\CONDITION 是判定结束得
   循环体
done
```
#### c语言风格的
(( exp1 ))
  while (( exp2 ));do
    COMMANDS
      (( exp3 ))
  done
### 如何生成列表：
 {1..100..2}
 `seq [起始数] [步进长度] [结束数]`
  `ls/etc`
### 系统定义变量：  
    自己一个在自己根目录下：cd    可以写进 .bash_profile  .bashrc
    系统全局 ：cd /etc/profile.d/env.sh    里面添加东西对全部用户都有影响。


echo -n 参数   不转行

echo "password" |passwd --stdin “user”   修改密码

shift 踢掉前一个参数；


-gt    是否大于           >
-ge   是否大于等于        >=
-eq   是否等于            =
-ne   是否不等于          ≠
-lt   是否小于            <
-le   是否小于等于        <=



# 实战

### 向系统中每个用户都问一声好： 并且输出shell 
```bash 
#!/bin/bash
#
#lines=`wc -l /etc/passwd |cut -d' ' -f1`
for i in `cat /etc/passwd|cut -d":" -f1`; do     //cut -d"指定字符" -f数字  
    for j in `cat /etc/passwd|cut -d":" -f7`;do  
              j=$j  
    done  
 echo Hello,$i ,I am xitong ,your shell is $j  
done
```
### 每个用户问好上同：不同方法
```bash
#!/bin/bash
#
lines=`cat /etc/passwd |wc -l`
for i in `seq 1 $lines` ; do
    echo hello "`cat /etc/passwd |head -"$i"|tail -1| cut -d":" -f1` ,your shell is `cat /etc/passwd |head -"$i"|tail -1| cut -d":" -f7`"
done
 echo this xitong is $lines
```



### 添加用户，删除用户：
```bash
if [ "$1" = '-s' ];then
        for i in `cat /data/script/script36/name.txt` ; do
            if id $i &> /dev/null;then
                echo $i  is exist
            else
                useradd $i
                echo $i |passwd --stdin $i>/dev/null
                echo $i succssful created 
            fi
        done
elif [ "$1" = '-c' ];then
        for j in `cat /data/script/script36/name.txt` ; do
            if id $j >/dev/null;then
                userdel -r $j
                echo $j delete succssful
            fi
        done
else
    echo " Please `basename $0` -c | -r "
```

### case 写菜单，选择一类的。可以写脚本在里面，输入什么就运行什么。
```bash
#!/bin/bash
#
case $1 in
  [0-9])
     echo it is "a digit"
     ;;
  [a-z])
    echo it is "lower"
     ;;
  [A..Z])
    echo it is "upper"
    ;;
  *)
    echo it is "other"
    ;;
esac
```bash
### 选择执行的命令服务，
#!/bin/bash
case $1 in
     start)
     echo "start server"
     ;;
  stop)
    echo "stop server"
     ;;
  statul)
    echo "statul server"
    ;;
  runing)
    echo "runing server"
    ;;
   *)
    echo "your `basename $0` start|stop|runing|statul"   \\提示输入的必须是以下四个
esac
```

### 用 for 的两个办法实现 1..100 相加。
```bash
sum=0
for i in echo {1..100};do
#    let sum+=i
    sum=$[sum+i]

done
 echo SUM is $sum

SUM=0
for ((k=1;k<=100;k++));do
 SUM=$[SUM+k]
done
  echo sum are $SUM
```
### for 循环打印9*9乘法表
```bash
for (( i=9;i<=9,0<i;i--));do
  for (( j=1;j<=i;j++));do
    echo -e "${j}x${i}=$[j*i]\t\c"
  done
  echo 
done
正着的
反正得

for (( i=1;i<=9;i++));do
  for (( j=1;j<=i;j++));do
    echo -e "${j}x${i}=$[j*i]\t\c"
  done
  echo 
done
```

### while实现1+100；
```bash
j=1
wsum=0
while [ $j -le 100 ];do
  wsum=$[$wsum+$j]
  let j++
done
  echo wsum am $wsum
```