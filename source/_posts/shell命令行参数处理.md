---
title: shell命令行参数处理
date: 2021-04-10T00:15:04+08:00
tags: [shell, codebase]
categories: shell
---
我使用Linux已有8年有余，经常会编写shell脚本进行自动化处理。然而，到目前为止，我依然不能像熟练使用C语言一样编写shell脚本。确实，我的主力编程语言是C语言，仅在做自动化脚本或者编写自动化测试用例的时候才会使用shell。另外一方面，shell脚本的语法的变种太多，例如，if语句在做字符串、数值、文件比较时的判断语句都相差很大；特殊符号多，`$#`、`$@`、`$?`等等，如果你是第一次接触shell脚本，必然会手足无措，更坑爹的是，这些特殊符号使用的场景相差很大，记忆负担真的大！！还有，shell脚本会依赖太多小程序，正如unix哲学所言：一个工具肩负单一使命，这些程序的各自用途、各自选项差异很大，你根本没有办法一下子就记住所有用法！！！

上面就是对自己写不好shell脚本的一些反思，破局还是有办法的。既然shell脚本的用法诡谲多变、充满奇淫技巧，依靠大脑的记忆肯定是不靠谱的，应该建立[codebase](https://github.com/system-thoughts/codebase/tree/master/shell)记录那些不太好记忆的shell语法、指令的典型应用示例。当忘记了相关的shell命令时，便可翻阅codebase唤醒代码记忆。当然，那些已深刻在你记忆中的知识就冗余无需记录。本篇文章是shell codebase的第一篇文章，主要介绍shell命令行参数的处理。

## 命令行参数基础
向shell脚本传输数据最基本的方法是使用命令行参数。传入的命令行参数通过位置参数(positional parameter)的特殊变量进行区分：`$0`表示程序名，`$1`、`$2`分别表示第一、第二个参数，依次类推。需要注意的是`$0`表示的是命令行中执行shell脚本的路径，使用相对路径和绝对路径，`$0`中的内容会相应改变。可使用`basename`提取文件名。
shell将空格作为参数的分割符，若参数中需要包含空格，则需要用引号（单引号、双引号均可）完整地包含参数。

有个小聪明啊，可以编写基于脚本名来执行不同功能的脚本:
```bash
# cat filename_as_func.sh 
#!/bin/bash
# implement the corresponding function according to the file name

name=$(basename $0)
if [ $name = "add" ]; then
	total=$[ $1 + $2 ]
elif [ $name = "mul" ]; then
	total=$[ $1 * $2 ]
fi

echo The caculation value is $total
```
将`filename_as_func.sh`链接到不同的文件名，查看运行结果：
```bash
# ln -s filename_as_func.sh add
# ln -s filename_as_func.sh mul
# ./add 2 3
The caculation value is 5
# ./mul 2 3
The caculation value is 6
```
检查预期的命令行参数是否存在是一种良好的编码风格，让脚本直接报错不可取。如下是检查第一个参数是否存在：
```bash
if [ -z "$1" ]; then
    echo parameter is needed!
    exit 1
fi
```

bash shell中有些特殊变量记录命令行参数。
`$#`表示命令行参数的个数。很直观地，我们会考虑借助`$#`表示最后一个命令行参数：`${$#}`。然而这种表示并不正确，不能在花括号中使用美元符，必须将美元符替换为感叹号，即`${!#}`。
```bash
[root@localhost parameters]# cat last_parameter.sh 
#!/bin/bash
# use $# represent the last parameter

echo The last parameter is ${!#}
[root@localhost parameters]# ./last_parameter.sh 1 2 3 4
The last parameter is 4
```

`$@`和`$*`特殊变量都能表示所有参数。
* `$*`会将命令行提供的所有参数当作单个单词进行访问。
* `$@`会将命令行上提供的所有参数当作同一个字符串中多个独立的单词。它允许你遍历所有值。
```bash
[root@localhost parameters]# cat all_parameter.sh 
#!/bin/bash
# test $* and $@ difference, they both represent all the parameters

count=1
for param in "$*"
do
	echo "\$* parameter #$count = $param"
	count=$[ $count + 1 ]
done

count=1
for param in "$@"
do
	echo "\$@ parameter #$count = $param"
	count=$[ $count + 1 ]
done
[root@localhost parameters]# ./all_parameter.sh  1 2 3 4
$* parameter #1 = 1 2 3 4
$@ parameter #1 = 1
$@ parameter #2 = 2
$@ parameter #3 = 3
$@ parameter #4 = 4
```

## 处理命令行选项
执行脚本通常会指定命令行选项，实际上，可以像处理命令行参数一样，处理命令行选项。我们先来看看最硬核的“手撕shell命令行选项”的处理方式。
```bash
[root@localhost parameters]# cat basic_options.sh 
#!/bin/bash
# handle options by hand

while [ -n "$1" ]
do
	case "$1" in
		-a) echo "Found the -a option" ;;
		-b) echo "Found the -b option" ;;
		-c) echo "Found the -c option" ;;
		*) echo "$1 is not an option" ;;
	esac
	shift
done
[root@localhost parameters]# ./basic_options.sh -a -b -c 1
Found the -a option
Found the -b option
Found the -c option
1 is not an option
```
`shift`命令会将每个参数变量减一。所以变量`$2`会移动到`$1`，而原来的`$1`会被删除，依次类推。注意：变量`$0`的值是不会改变，即程序名不会改变。所以通过`while`循环配合`shift`命令便能够处理所有命令行参数。通过`case`语句判断命令行选项，并做相应处理。

上面的脚本可以识别选项，但同时使用选项和参数，则对参数无法很好地处理。Linux中处理这个问题的标准方法是使用特殊字符（双破折线--）将选项和参数分开，特殊字符会告诉脚本，选项何时结束以及普通参数何时开始。
```bash
[root@localhost parameters]# cat extract_parameters.sh 
#!/bin/bash
# extract options and parameters

while [ -n "$1" ]
do
	case "$1" in
		-a) echo "Found the -a option" ;;
		-b) echo "Found the -b option" ;;
		-c) echo "Found the -c option" ;;
		--) shift
		    break ;;
		*) echo "$1 is not an option"	
	esac
	shift
done

count=1
for param in $@
do
	echo "parameter #$count is $param"
	count=$[ $count + 1 ]
done
[root@localhost parameters]# ./extract_parameters.sh -a -b -c -- 1 2 3
Found the -a option
Found the -b option
Found the -c option
parameter #1 is 1
parameter #2 is 2
parameter #3 is 3
```
当脚本遇到双破折线时，就停止处理选项，将剩下的参数都作为命令行参数。然而，此种方式将命令行选项和参数泾渭分明。如果命令行选项会带有额外的参数，此种方式便无法区分，需要手动处理选项参数：
```bash
[root@localhost parameters]# cat basic_option_parameter.sh 
#!/bin/bash
# extract option parameter by hand

while [ -n "$1" ]
do
	case "$1" in
		-a) echo "Found the -a option" ;;
		-b) param="$2"
			echo "Found the -b option, with parameter value $param"
			shift ;;
		-c) echo "Found the -c option" ;;
		--) shift
		    break ;;
		*) echo "$1 is not an option"	
	esac
	shift
done

count=1
for param in $@
do
	echo "parameter #$count is $param"
	count=$[ $count + 1 ]
done
[root@localhost parameters]# ./basic_option_parameter.sh -a -b 2 -c -- 56 78
Found the -a option
Found the -b option, with parameter value 2
Found the -c option
parameter #1 is 56
parameter #2 is 78
```
当前shell脚本已经具备处理命令行选项的基本能力，但是还有一些限制，如无法区分合并的命令行选项。另外，没有用户愿意通过输入"--"区分选项和参数，这个工作最好由shell脚本自动完成。下面介绍的`getopt`命令就是为了摆脱这种最硬核的选项解析方式。

## getopt命令
`getopt`命令是处理命令行选项和参数的一个非常强有力的工具，在CentOS中，其由`util-linux`提供。`getopt`命令参数可以分为两部分：1.命令行选项`[options]` 2. `getopt`待解析的命令行参数`parameters`，`parameters`是从第一个非`getopt`命令行选项开始的，或者在`--`之后。根据`getopt`命令第一部分的状态，`getopt`的调用方式可以分为三种：
1. 无命令行选项，此种选项解析最为简单，也是最常用的，但是只能解析短选项：
```bash
getopt optstring parameters
```
* `parameters`：getopt要解析的命令行参数
```bash
# getopt "ab:cd" -a 1 -b 2 -cd 3 4
 -a -b 2 -c -d -- 1 3 4
```
2. 有命令行选项，但是没有`-o|--options`选项，`-o`选项比较特殊，用来指定短选项(shortopts)：
```bash
getopt [options] [--] optstring parameters
```
* options表示所有的非`-o`选项，如果没有`--`，则第一个非`getopt`选项的参数就是shortopts，随后便是待解析参数。
```bash
[root@localhost ~]# getopt "ab:cd" -c 1 -b 2 -ade 3 4
getopt: invalid option -- 'e'
 -c -b 2 -a -d -- 1 3 4
[root@localhost ~]# getopt -q "ab:cd" -c 1 -b 2 -ade 3 4
 -c -b '2' -a -d -- '1' '3' '4'
[root@localhost ~]# echo $?
1
[root@localhost ~]# getopt -q -- "ab:cd" -c 1 -b 2 -ade 3 4
 -c -b '2' -a -d -- '1' '3' '4'
[root@localhost ~]# echo $?
1
```
如果指定了一个不存在`optstring`的选项，默认情况下，`getopt`会报错，通过`-q`选项会忽略这条错误信息，但是`getopt`还是会返回错误值。上面例子的最后两条指令表明了shortopts的位置。这种隐式指明shortopts位置的方式在有`-l|--longoptions`参数的时候很容易造成误解。同shortopts指定参数的方式，唯一不同的是，longopts的每个参数通过逗号分隔。
```bash
[root@localhost ~]# getopt -l extract:,tar:,help,directory -- file --extract a.tar
 --extract 'a.tar' --
[root@localhost ~]# getopt -l extract:,tar:,help,directory -- file --extract a.tar -f -i abc
 --extract 'a.tar' -f -i -- 'abc'
 ```
 上面这个示例，我们打算只定义长命令行选项。然而第一条shell命令未解析到参数`file`。再看看后一条命令，大概就心里有数了。如果`getopt`命令带有选项，但是未带有`-o`选项，`getopt`会将命令第二部分的第一个参数即`file`作为`shortopts`。这说明`getopt`默认总是会解析短选项，其必须指定`shortopts`。然而，这种隐式的指定，对于命令的理解极其不友好。
 3. 若要指定长命令选项，最好通过`-o|--options`显式地指明短命令选项：
```bash
getopt [options] -o|--options optstrin [options] [--] parameters
 ```
还是上面的例子，但同时指明短命令选项：
```
[root@localhost ~]# getopt -o e:t:hd -l extract:,tar:,help,directory -- file --extract a.tar
 --extract 'a.tar' -- 'file'
[root@localhost ~]# getopt -o '' -l extract:,tar:,help,directory -- file --extract a.tar
 --extract 'a.tar' -- 'file'
 ```
可见，即便你不想指定短选项，显式地通过将`-o`选项指定为空。也能够正确地解析出参数`file`。

使用`getopt`处理脚本的命令行参数：
```bash
[root@localhost parameters]# cat basic_getopt.sh 
#!/bin/bash
# use getopt handle parameters

set -- $(getopt -q "ab:c" "$@")
while [ -n "$1" ]
do
	case "$1" in
		-a) echo "Found the -a option" ;;
		-b) param="$2"
			echo "Found the -b option, with parameter value $param"
			shift ;;
		-c) echo "Found the -c option" ;;
		--) shift
		    break ;;
		*) echo "$1 is not an option"	
	esac
	shift
done

count=1
for param in $@
do
	echo "parameter #$count is $param"
	count=$[ $count + 1 ]
done
[root@localhost parameters]# ./basic_getopt.sh -c 1 -b 2 -ad 3 4
Found the -c option
Found the -b option, with parameter value '2'
Found the -a option
parameter #1 is '1'
parameter #2 is '3'
parameter #3 is '4'
```
其中最为关键的是`set`与`getopt`的配合使用。前面，我们已经提到`--`是选项和参数的分界符，`--`之后都代表`set`命令的参数，即便其中包含`-`，也不会被识别为`set`命令的选项。上述set命令会将当前环境变量`$@`设置为`getopt -q "ab:c" "$@"`的输出。那么，这条`set`命令就是为了用`getopt`格式化后的命令行参数来替换原始的命令行参数。后续while循环处理的便是格式化的命令行参数即：
```bash
[root@localhost parameters]# getopt -q "ab:c" -c 1 -b 2 -ad 3 4
 -c -b '2' -a -- '1' '3' '4'
 ```

## getopts命令
bash shell包含了`getopts`命令。与`getopt`将命令行上找到的选项和参数处理后只生成一个输出不同。每次调用`getopts`，它只处理一个命令行上检测到的参数。处理完所有参数后，它会退出并返回一个大于零的退出状态码。因此，`getopts`非常适合用于解析命令行所有参数的循环中。
`getopts`命令的格式如下：
```bash
getopts optstring variable
```
`optstring`同`getopt`命令的`shortopts`。若`getopts`要忽略错误信息，可以在`optstring`之前加上冒号。`variable`表示待解析的命令行参数。

`getopts`会用到两个环境变量。如果选项后面跟参数，`OPTARG`环境变量保存该参数值。`OPTIND`环境变量保存了参数列表中`getopts`正在处理的参数位置。
```bash
[root@localhost parameters]# cat getopts_OPTIND.sh 
#!/bin/bash
# processing options and prameters with getopts, show OPTIND and shift combination to get parameters

while getopts :ab:cd opt
do
	case "$opt" in
		a) echo "Found the -a option" ;;
		b) echo "Found the -b option, with value $OPTARG" ;;
		c) echo "Found the -c option" ;;
		d) echo "Found the -d option" ;;
		*) echo "Unknown option $opt" ;;
	esac
done

shift $[ $OPTIND - 1 ]

count=1
for param in  "$@"
do
	echo "Parameter $count: $param"
	count=$[ $count + 1 ]
done
[root@localhost parameters]# ./getopts_OPTIND.sh -a -btest1 -d -f "test2 test3" 
Found the -a option
Found the -b option, with value test1
Found the -d option
Unknown option ?
Parameter 1: test2 test3
```
由上面的示例可知，`getopts`处理每个选项时，它会将`OPTIND`环境变量的值增1。在`getopts`完成处理后，可以将`OPTIND`和`shift`命令一起使用移动参数。
与`getopt`命令不同的是：`getopts`解析命令行选项时，会移除开头的破折号。它能够将命令行上找到的所有未定义的选项统一输出问号。同时`getopts`不支持长选项的解析。
但是`getopts`命令并不灵活：
```bash
[root@localhost parameters]# ./getopts_OPTIND.sh -a -btest1 -d  "test2 test3"  -c
Found the -a option
Found the -b option, with value test1
Found the -d option
Parameter 1: test2 test3
Parameter 2: -c
```
如果参数出现在选项之前，`getopts`便不能够解析到选项了。

因此`getopt`提供了较为强大且定制的功能，`getopts`提供了快捷的命令行参数选项解析功能，但并不灵活。

## 解析选项冲突
同一命令的不同选项存在冲突语义的情况，例如`tar`命令的`-x`选项表示解压tar包，`-t`选项表示打包操作。显然，这两个选项的语义互相冲突。`getopt`和`getopts`并未定义选项之间的关系，因此我们必须手动解析选项之间的关系，可参考下面的例子：
```bash
[root@localhost parameters]# cat option-relation.sh 
#!/bin/bash
# handle options' relationship

set -- $(getopt -q "t:x:h" "$@")
while [ -n "$1" ]
do
	case "$1" in
		-t) tar=true
			tar_param="$2"
			shift ;;
		-x) extract=true
			extract_param="$2"
			shift ;;
		--) shift
			break ;;
		*) echo "$1 is unrecognized"
	esac
	shift
done

if [[ $tar == true && $extract == true ]]; then
	echo "Cannot specify -t -x options at the same time"
	exit 1
elif [[ $tar == true ]]; then
	echo "Exec $(basename $0) -t $tar_param ..."
elif [[ $extract == true ]]; then
	echo "Exec $(basename $0) -x $extract_param ..."
fi

count=1
for param in $@
do
	echo "parameter #$count is $param"
	count=$[ $count + 1 ]
done
[root@localhost parameters]# ./option-relation.sh -x a.tar -t file file2
Cannot specify -t -x options at the same time
[root@localhost parameters]# ./option-relation.sh -x a.tar  file2
Exec option-relation.sh -x 'a.tar' ...
parameter #1 is 'file2'
[root@localhost parameters]# ./option-relation.sh -t file file2
Exec option-relation.sh -t 'file' ...
parameter #1 is 'file2'
```
这不一定是最好的方法，但是代码较为清晰。前半部分解析选项，标记相应的flag。然后根据flag情况判断相应的依赖关系，进行实际的命令行选项处理。
