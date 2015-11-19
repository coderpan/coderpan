title: Makefile实例解析
date: 2015-11-18 17:56:56
tags: makefile
categories: Docs
description: Introduce The Makefile Simply !
feature: Makefile.jpg
toc: true
---

对于从事linux下C++开发的同学来说，makefile方面的知识是必不可少的。本文将选取一个实例来讲解makefile的运用方法,可能存在一些错误的地方，希望看到的同学批评指正，也借此抛砖引玉了：

一般来说，一个项目的后台服务部门至少包含一个公共的makefile文件，其中包含平台属性定义（比如32位/64位的判断等）、公共库目录（公共动态库、静态库的头文件目录、.o/.so文件生成目录等）、通用操作定义（all、clean、release等）...

> {% textcolor info %}本文适合对 Makefile 接触不多的同学阅读，其中涉及到的知识面比较窄，后面会慢慢完善！{% endtextcolor %}


## 项目公共Makefile ##

以下是我选取的项目公共makefile文件，讲述将以注释的形式穿插在代码中，过程中会顺带讲些与makefile相关的内容：

<!-- more -->

```Makefile
#strip函数显然是去空格，这句是获取主机CPU位数，用于编译参数 -m32（编译32位版本）、-m64（编译64位版本）,
#可以shell执行下uname -m就明白。
PLATFORM                := $(strip $(shell echo `uname -m`))

#MFLAGS将在子服务的makefile中定义,再include这个公共makefile即可
#如果外面没有定义，将使用uname -m得到的位数编译，这样的好处是可以指定位数编译
MFLAGS                  := $(strip $(MFLAGS))
ifeq ($(MFLAGS),)
ifeq ($(PLATFORM),x86_64)
        MFLAGS          := 64
else
        MFLAGS          := 32
endif
endif

#要编译的目标（可执行程序、静态库文件、动态库文件等）在外层makefile中定义
TARGET                  := $(strip $(TARGET))

#定义版本的宏，通过gcc/g++的-D参数传入，代码中就可以使用这个宏用于区分版本
VERSION_MAJOR  := 1
VERSION_MINOR  := 0
VERSION_PATCH  := 0
VERSION_PATCH_MINOR := 0
HAF_VERSION    := $(VERSION_MAJOR).$(VERSION_MINOR).$(VERSION_PATCH).$(VERSION_PATCH_MINOR)

#定义编译器，后面会介绍gcc和g++的使用场景
CC              = gcc
CXX             = g++

#首选的一些编译参数：
#-g: 编译出的目标文件带源文件的调试信息，用于gdb调试
#-fPIC: 告诉编译器产生与位置无关代码(Position-Independent Code)，则产生的代码中，
#       没有绝对地址，全部使用相对地址，故而代码可以被加载器加载到内存的任意位置，
#       都可以正确的执行。这正是共享库.so所要求的，共享库被加载时，在内存的位置不是固定的。
#-Wno-deprecated: 告诉编译器不要产生deprecated的警告,那么什么是deprecated警告呢？如果代码中
#       使用了编译器的老特性（在未来会被抛弃的特性）时，编译产生的告警就是deprecated警告，举个例子：
#       一个返回值为void*类型的函数func(),g++允许这样使用(struct AA) *pt = func();这就会产生
#       deprecated警告，像这样的老特性还有不少，如果编译时不希望有这样的Warning，可以加上这个参数。
#-Wall: 打开警告开关(waring all)，-O代表默认优化，可选：-O0不优化，-O1低级优化，-O2中级优化，
#       -O3高级优化，-Os代码空间优化,一般这两个参数同时使用，有什么好处呢？举个很简单的例子：
#       int a;print a;使用没有初始化的变量时，编译优化就会产生warning了
CFLAGS          += -g -fPIC -Wno-deprecated -Wall -O -DHAF_VERSION=\"$(HAF_VERSION)\"

#TOPDIR为工程根目录，尽量在外层makefile中以相对路径的形式定义，这样比较灵活
#定义根目录是为了编译时嵌入include目录，lib目录等,一般情况下这些目录就在根目录
INCLUDE         += -I${TOPDIR}/include
HAFLIB_PATH     := ${TOPDIR}/lib/

#定义编译时需包含的库文件参数，比如需要包含/AAA/lib/libBBB.a，编译参数加上 -L/AAA/lib/ -lbbb即可
LIB             += -L${HAFLIB_PATH} -lpthread

#接下来就是提取源文件（.cpp .c等）列表，目标文件（.o文件）列表，依赖文件（.depend文件）列表了,
#目的是为了拼接在编译参数后面编译，比如我们熟知的gcc test.c -o test，这里会用到makefile函数：
#wildcard: 提取当前目录这两个后缀的文件列表
#sort: 按文件名排序
#patsubst: 替换，将上一步产生的文件列表中所有的.c替换成.o，所有的.cpp替换成.o，形成目标文件列表
#foreach:  makefile语法中的for循环，举个简单的例子：
#          names := a b c d
#          files := $(foreach n,$(names),$(n).o)
#          这样files就等于a.o b.o c.o d.o
#dir和nodir: 分别取/dir/和filename
LOCAL_SRC       += $(sort $(wildcard *.cpp *.c))
LOCAL_OBJ       += $(patsubst %.cpp,%.o, $(patsubst %.c,%.o, $(LOCAL_SRC)))
DEP_FILE        := $(foreach obj, $(LOCAL_OBJ), $(dir $(obj)).$(basename $(notdir $(obj))).d)

#----------------------------------------------------------------------------------
#这是一个函数，用于将目标文件.a移动到根目录下的lib目录,熟悉shell编程的同学应该可以看的明白，
#这里就不作讲解了。
copyfile = if test -z "$(APP)" || test -z "$(TARGET)"; then \
               echo "['APP' or 'TARGET' option is empty.]"; exit 1; \
            else \
                if test ! -d $(2); then \
                    echo "[No such dir:$(2), now we create it.]";\
                    mkdir -p $(2);\
                fi; \
                echo "[Copy file $(1) -> $(2)]"; \
                cp -v $(1) $(2); \
            fi;
#----------------------------------------------------------------------------------

#phony单词翻译是“假的，赝品"，顾名思义，跟在后面的都是假的目标，只是一个标识
#如果不加phony的话，那么就把clean当成了目标文件，紧跟其后的"："后面并没有依赖文件，
#就会导致每次make clean都会提示目标是最新的，无法清除文件,反之，如果我们加上了,这次额标识
#就不会当成目标文件，也就可以清除文件了。
.PHONY: all clean release

#为什么makefile里面都会有一个all呢？因为默认执行make命令的时候，其实就是make all。
#all后面所有的依赖文件以及 依赖文件所依赖的文件 都会按照顺序递归生成
all : 
    $(LOCAL_OBJ) $(DEP_FILE) $(TARGET)

#filter函数：顾名思义就是过滤了，将TARGET里的.a文件过滤出来单独生成，因为普通的可执行文件和.a以及.so等
#生成方式都会不同，所以要单独处理。
#ar是将.o文件链接生成静态库的命令。
#$@是本次目标生成动作中所有的目标文件，对应的$^是所有依赖文件，$<是第一个依赖文件
$(filter %.a,$(TARGET)) : $(LOCAL_OBJ)
    ar r $@ $(LOCAL_OBJ)

#-shared是生成共享库.so必须参数，如果指定了这个参数，一定要加上-fPIC，原因前面已讲述
#.so的链接我们一般采用gcc的链接器来完成
$(filter %.so,$(TARGET)) : $(LOCAL_OBJ)
    $(CC) -m$(MFLAGS) -shared -o $@ $(LOCAL_OBJ) $(LIB)

#上面讲了filter，对应filter-out的意思就呼之欲出了：过滤之后剩下的文件
#这里一般对应普通的可执行文件的编译
$(filter-out %.so %.a,$(TARGET)) : $(LOCAL_OBJ)
    $(CXX) -m$(MFLAGS) $(CFLAGS) -o $@ $^ $(INCLUDE) $(LIB)

clean:
    rm -vf $(LOCAL_OBJ) $(TARGET) $(DEP_FILE)

#call调用makefile自定义函数copyfile
release:
    @$(call copyfile, *.a, $(HAFLIB_PATH))

#嵌入依赖文件，大家打开依赖文件都可以看出来,里面是每个目标文件的所有依赖文件列表
#后面会讲到.d文件的生成方法，这里先说下为什么要生成并嵌入.d文件：
#C源码的开头经常有一系列被包含的头文件，例如 stdio.h。
##include <stdio.h>#include "foo.h"int main(....)
#如果你的 foo.h 被改变之后，要确保这个文件也会被重新编译，就要在你的 Makefile 这样写：
#foo: foo.c foo.h
#当你的项目变得越来越大，你自己的头文件越来越多的时候，要追踪所有这些头文件和所有依赖它的
#文件会是一件痛苦的事情。如果你改变了其中一个头文件，却忘了重新编译所有依赖它的源文件，结果会让我们失望。
#这个时候就需要用到.d文件了，该文件里列出来所有源文件需要包含的头文件，并按照makefile的语法组成依赖关系，
#这样我们嵌入进来之后，就可以达到“foo: foo.c foo.h”这样的目的了。
ifneq ($(DEP_FILE),)
-include $(DEP_FILE)
endif

#这里即是生成依赖文件.depend文件：
#gcc -MM即是生成依赖关系文件的方式
#第三行就是gcc -MM test.cpp将生成的“test.o: test.cpp ../../include/test.h“写入.test.d中。
.%.d: %.cpp %.c
    @echo "update $@ ..."; \
    $(CC) $(INCLUDE) -MM $< >> $@;

#下面编译cpp和c文件分别用g++和gcc编译
#注意：加-c选项是告诉编译器，只编译成指定的.o文件即可，不需要链接
%.o: %.cpp
    $(CXX) -m$(MFLAGS) $(CFLAGS) $(INCLUDE) -o $@ -c $<

%.o: %.c
    $(CC) -m$(MFLAGS) $(CFLAGS) $(INCLUDE) -o $@ -c $<
```

## 子服务Makefile
上面讲完公共makefile之后，这里将附上子服务的makefile，看是如何引用公共makefile的，
这样分开之后的好处不言而喻，每增加一个子服务只需要在外层makefile定义一些独有的变量即可：

```Makefile
#include相当于把公共makefile拷贝一份放到这里，先定义这些变量再include公共makefile，
#定制子服务独有的makefile
APP            := LEARNING
TOPDIR         := ../..

#编译要用的静态库
INCLUDE +=
LIB     += -lutil

TARGET := test

include ${TOPDIR}/makefile
```

</br>
#### &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;转载本站文章请注明作者和出处 __码畔-coderpan__，请勿用于任何商业用途
