title: Makefile实例解析
date: 2015-11-18 17:56:56
tags: makefile
---

对于从事linux下C++开发的同学来说，makefile方面的知识是必不可少的。本文将选取一个实例来讲解makefile的运用方法,可能存在一些错误的地方，希望看到的同学批评指正，也借此抛砖引玉了：

一般来说，一个项目的后台服务部门至少包含一个公共的makefile文件，其中包含平台属性定义（比如32位/64位的判断等）、公共库目录（公共动态库、静态库的头文件目录、.o/.so文件生成目录等）、通用操作定义（all、clean、release等）...

以下是我选取的项目公共makefile文件，讲述将穿插在代码中，过程中会顺带讲些与makefile相关的内容：

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

TARGET                  := $(strip $(TARGET))
SUFFIX                  := $(suffix $(TARGET))

VERSION_MAJOR  := 1
VERSION_MINOR  := 0
VERSION_PATCH  := 0
VERSION_PATCH_MINOR := 0
HAF_VERSION    := $(VERSION_MAJOR).$(VERSION_MINOR).$(VERSION_PATCH).$(VERSION_PATCH_MINOR)

HAFLIB_PATH     := ${TOPDIR}/lib/
LIB             += -L${HAFLIB_PATH} -lpthread

INCLUDE         += -I${TOPDIR}/include

LOCAL_SRC       += $(sort $(wildcard *.cpp *.c))
LOCAL_OBJ       += $(patsubst %.cpp,%.o, $(patsubst %.c,%.o, $(LOCAL_SRC)))
DEP_FILE        := $(foreach obj, $(LOCAL_OBJ), $(dir $(obj)).$(basename $(notdir $(obj))).d)

CC              = gcc
CXX             = g++
CFLAGS          += -g -fPIC -Wno-deprecated -Wall -DHAF_VERSION=\"$(HAF_VERSION)\"
```
