---
title: NCCL源码学习： Makefile
date: 2024-01-06 13:55:14
tags:
  - nccl
  - gpu
  - network
---

[NCCL Makefile](https://github.com/NVIDIA/nccl/blob/master/src/Makefile)，`make build`生成目标动态链接库、静态链接库以及需要安装到系统目录中的头文件。

# include xxx.mk

在Makefile的一开始使用`include`指令包含了两个`.mk`文件

```makefile
include ../makefiles/common.mk
include ../makefiles/version.mk
```

在 Makefile 中，`include` 是一个指令，用于包含其他 Makefile 或规则文件。当使用 include 指令时，Makefile 会读取并解析指定的文件，并将其内容合并到当前的 Makefile 中，这样可以将多个 Makefile 或规则文件组织在一起，使得整个构建过程更模块化和可维护。

<!--more-->

在这个源码中，`version.mk`文件定义了与项目版本号相关的变量

```makefile
##### version
NCCL_MAJOR   := 2
NCCL_MINOR   := 19
NCCL_PATCH   := 4
NCCL_SUFFIX  :=
PKG_REVISION := 1
```

# wildcard

接着定义了需要export的两个头文件，以及一些库源文件，其中定义库源文件时用到了`wildcard`函数

```makefile
##### src files
INCEXPORTS  := nccl.h nccl_net.h
LIBSRCFILES := \
	bootstrap.cc channel.cc collectives.cc debug.cc enqueue.cc group.cc \
	init.cc init_nvtx.cc net.cc proxy.cc transport.cc \
	$(wildcard graph/*.cc) \
	$(wildcard misc/*.cc) \
	$(wildcard transport/*.cc)
```

在 Makefile 中，`wildcard` 是一个函数，用于在目录中匹配文件名模式，并返回匹配的文件列表。

# abspath

接着定义了最终要生成的动态链接库、静态链接库、配置信息文件名，以及build的头文件目录、库文件目录、目标文件目录、配置文件目录等，其中在定义构建目录时，使用了`abspath`函数

```makefile
##### lib files
LIBNAME     := libnccl.so
STATICLIBNAME := libnccl_static.a
##### pkgconfig files
PKGCONFIGFILE := nccl.pc
##### dirs
BUILDDIR ?= $(abspath ../build)
INCDIR := $(BUILDDIR)/include
LIBDIR := $(BUILDDIR)/lib
OBJDIR := $(BUILDDIR)/obj
PKGDIR := $(BUILDDIR)/lib/pkgconfig
```

`abspath`函数会将文件路径转换成绝对路径。

# 分支语句ifeq

接着判断用户是否传入了CUDARTLIB变量的定义，如果是默认值cudart_static，则将enhcompat.cc加入到变量LIBSRCPILES中，这段逻辑使用Makefile的`ifeq`

```makefile
##### target files
CUDARTLIB  ?= cudart_static

ifeq ($(CUDARTLIB), cudart_static)
	LIBSRCFILES += enhcompat.cc
endif
```

在 Makefile 中，`ifeq` 是一个条件判断指令，用于在构建过程中根据条件执行不同的操作。

# 动态链接库编译选项

接着使用`%`通配符定义了目标头文件、目标动态链接库、每个目标文件所依赖的文件、动态链接编译选项等变量。

```makefile
INCTARGETS := $(INCEXPORTS:%=$(INCDIR)/%)
LIBSONAME  := $(LIBNAME:%=%.$(NCCL_MAJOR))
LIBTARGET  := $(LIBNAME:%=%.$(NCCL_MAJOR).$(NCCL_MINOR).$(NCCL_PATCH))
STATICLIBTARGET := $(STATICLIBNAME)
PKGTARGET  := $(PKGCONFIGFILE)
LIBOBJ     := $(LIBSRCFILES:%.cc=$(OBJDIR)/%.o)
DEPFILES   := $(LIBOBJ:%.o=%.d)
LDFLAGS    += -L${CUDA_LIB} -l$(CUDARTLIB) -lpthread -lrt -ldl

DEVMANIFEST := $(BUILDDIR)/obj/device/manifest
```

在编译源代码时，`-L` 和 `-l` 是常用的编译选项，用于指定编译器在链接时搜索库文件的路径和链接库文件。`-L` 选项用于指定编译器在链接时搜索库文件的路径，后面需要跟着库文件所在的目录路径。`-l`选项用于指定编译器在链接时要使用的库文件，后面跟着库文件的名称（不包括文件名前缀lib和文件扩展名）。

编译器解析`-l`后面的链接库时，会根据指定的名称自动搜索库文件，首先在`-L`指定的目录中查找，如果没找到会去系统默认查找目录或LD_LIBRARY_PATH环境变量定义的路径下查找，如果没有指定`-static`选项，会优先查找动态链接库。

顺带说一下`DEVMANIFEST`文件：通常用于描述设备驱动程序的清单或配置信息，提供了关于设备驱动程序的元数据、配置选项和相关文件的详细信息。使用 `DEVMANIFEST` 文件，可以更轻松地管理和配置设备驱动程序，确保其正确安装、编译和运行。

# 强制重新构建目标

这一段例子让目标依赖于`ALWAYS_REBUILD`，而`ALWAYS_REBUILD`是一个空目标，从而实现永远重新构建DEVMANIFEST的目的。

```makefile
##### rules
build : lib staticlib

lib : $(INCTARGETS) $(LIBDIR)/$(LIBTARGET) $(PKGDIR)/$(PKGTARGET)

staticlib : $(LIBDIR)/$(STATICLIBTARGET)

$(DEVMANIFEST): ALWAYS_REBUILD $(INCTARGETS)
	$(MAKE) -C ./device

# Empty target to force rebuild
ALWAYS_REBUILD:
```

# -include指令

源码里使用`-include`包含了前面定义的`.d`依赖文件

```makefile
-include $(DEPFILES)
$(LIBDIR)/$(LIBTARGET) $(LIBDIR)/$(STATICLIBTARGET) : $(LIBOBJ)
```

在 Makefile 中，`-include` 是一个特殊的指令，用于包含其他文件，通常是用于包含依赖关系文件。在大型项目中，手动维护和更新依赖关系非常复杂和繁琐。为了解决这个问题，可以使用自动生成的依赖关系文件，例如使用 `gcc -M` 选项生成的文件。这些文件列出了源文件与其依赖项之间的关系。然而，如果依赖关系文件不存在或无效，使用普通的 `include` 指令将导致 `make` 命令失败并停止构建。为了避免这种情况，可以使用 `-include` 指令，也可以写作 `--include`。

`-include` 指令会在 Makefile 执行期间读取并包含指定的文件，即使这些文件不存在或无效也不会导致错误。这样，如果依赖关系文件不存在，或者在构建系统中生成依赖关系文件的过程中出现问题，make 命令仍然可以继续执行。

# 使用.in 模板文件生成.h文件

在下面这个片段中，对于`nccl.h`目标来说，它依赖于`nccl.h.in`和`veriosn.mk`文件，其中`.in`文件作为模板文件，用于生成最终的头文件。`.in` 文件包含占位符和变量，代表需要在生成过程中替换的值。在这个例子中，使用`sed`来解析 `.in` 文件，将占位符替换为实际的值，并生成最终的头文件`nccl.h`。使用 `.in` 文件的好处是根据不同环境生成不同版本的头文件。

```makefile
$(INCDIR)/nccl.h : nccl.h.in ../makefiles/version.mk
	@$(eval NCCL_VERSION := $(shell printf "%d%02d%02d" $(NCCL_MAJOR) $(NCCL_MINOR) $(NCCL_PATCH)))
	mkdir -p $(INCDIR)
	@printf "Generating %-35s > %s\n" $< $@
	sed -e "s/\$${nccl:Major}/$(NCCL_MAJOR)/g" \
	    -e "s/\$${nccl:Minor}/$(NCCL_MINOR)/g" \
	    -e "s/\$${nccl:Patch}/$(NCCL_PATCH)/g" \
	    -e "s/\$${nccl:Suffix}/$(NCCL_SUFFIX)/g" \
	    -e "s/\$${nccl:Version}/$(NCCL_VERSION)/g" \
	    $< > $@
```

## eval 函数

eval函数用于在运行时动态为一个变量复制，因为有时候对一个变量的赋值要在运行时才能确定。

## @ 抑制输出

`@$(eval ...` 中的`@` 符号告诉 make 工具不要输出该命令的执行过程，在终端上不会显示命令本身，而只会显示它的执行结果。通过使用`@` 前缀，可以使得 Makefile 的输出更加清晰和简洁，只显示与构建过程相关的信息，而不会混杂着大量的命令执行输出。

## $< 和 $@ 自动变量

`$<` 和 `$@` 是自动变量，用于表示规则中的依赖文件和目标文件的名称，例子中将依赖文件按照左对齐、至少占据35个字符的字符串格式输出。

- `$<`  规则的第一个依赖文件的名称，即`nccl.h.in`
- `$@` 规则的目标文件的名称，即`nccl.h`

## sed命令

`sed`是stream editor的缩写，用于在文本中替换字符串。例子中将nccl.h.in的占位符替换成正确的版本号，并重定向输出到nccl.h文件中。注意：Makefile中使用`$$`转义`$`并传给命令`sed`，因为Makefile中`$`有特殊含义

# 生成动态链接库

动态链接库`libnccl.so`依赖于`.o`文件，如果需要更新`.so`文件，会调用C编译器将`.o`文件与需要的库进行链接，最终生成目标动态链接库。

```makefile
$(LIBDIR)/$(LIBTARGET): $(LIBOBJ) $(DEVMANIFEST)
	@printf "Linking    %-35s > %s\n" $(LIBTARGET) $@
	mkdir -p $(LIBDIR)
	$(CXX) $(CXXFLAGS) -shared -Wl,--no-as-needed -Wl,-soname,$(LIBSONAME) -o $@ $(LIBOBJ) $$(cat $(DEVMANIFEST)) $(LDFLAGS)
	ln -sf $(LIBSONAME) $(LIBDIR)/$(LIBNAME)
	ln -sf $(LIBTARGET) $(LIBDIR)/$(LIBSONAME)
```

gcc创建共享库的编译选项：最终生成`build/lib/libnccl.so.2.19.4`

- `CXX` Makefile中预定义的变量，表示当前使用的C编译器，通常为`g++`
- `CXXFLAGS` 一些需要构建共享库的标志位
	- `-fPIC` Position Independent Code，告诉编译器生成与位置无关的代码。
	- `-fvisibility=hidden` 控制符号的可见性，默认情况下，编译器将所有符号都导出为公共符号对外可见。使用`-fvisibility=hidden`选项可以限制库的可见性，只有明确标记为公共的符号才会对外可见，其他符号将被隐藏。
	- `-shared` 是一个链接器选项，用于告诉链接器生成一个共享库而不是可执行文件。
	- `-Wl,--no-as-needed` 是一个链接器选项，用于告诉链接器不要自动丢弃未使用的共享库依赖项。默认情况下，链接器会尽可能地删除未使用的共享库依赖项，以减少最终生成的共享库文件大小。
	- `-Wl,-soname,$(LIBSONAME)`是一个链接器选项，用于指定生成的共享库的 soname，在运行时，程序将使用指定的 soname来查找和加载共享库，在例子里是`libnccl.so.2`。
	- `$$(cat $(DEVMANIFEST))` 转义之后相当于`$(cat $(DEVMANIFEST))`，把`DEVMANIFEST`文件中的内容作为文件列表传递给`gcc`，比如`file1.c file2.c file3.c`。

创建符号链接：例子创建链接`build/lib/libnccl.so`，指向`libnccl.so.2`；以及`build/lib/libnccl.so.2`，指向`libnccl.so.2.19.4`

# 生成静态链接库

接着使用`ar`命令创建一个静态库（archive library）。

```makefile
$(LIBDIR)/$(STATICLIBTARGET): $(LIBOBJ) $(DEVMANIFEST)
	@printf "Archiving  %-35s > %s\n" $(STATICLIBTARGET) $@
	mkdir -p $(LIBDIR)
	ar cr $@ $(LIBOBJ) $$(cat $(DEVMANIFEST))
```

# 生成pkg-config文件

接着使用`.in`模板文件生成`.pc`文件，使用的方法和前面说的生成`nccl.h`方式类似。生成的 `.pc` 文件用于描述库的名称、版本、依赖关系、头文件路径、库路径、安装路径以及编译和链接选项等。

```makefile
$(PKGDIR)/nccl.pc : nccl.pc.in
	mkdir -p $(PKGDIR)
	@printf "Generating %-35s > %s\n" $< $@
	sed -e 's|$${nccl:Prefix}|\$(PREFIX)|g' \
	    -e "s/\$${nccl:Major}/$(NCCL_MAJOR)/g" \
	    -e "s/\$${nccl:Minor}/$(NCCL_MINOR)/g" \
	    -e "s/\$${nccl:Patch}/$(NCCL_PATCH)/g" \
	    $< > $@
```

注意：由于第一条`sed`命令替换的字符串本身带有`/`字符，因此可以使用`|`作为sed命令分隔符，避免对`/`进行转义

# install拷贝文件

以下目标的执行命令中，使用`install`命令复制文件，`-m 644`设置文件权限，在这里文件权限被设置为`-rw-r--r--`。

```makefile
$(INCDIR)/%.h : %.h
	@printf "Grabbing   %-35s > %s\n" $< $@
	mkdir -p $(INCDIR)
	install -m 644 $< $@

$(INCDIR)/nccl_%.h : include/nccl_%.h
	@printf "Grabbing   %-35s > %s\n" $< $@
	mkdir -p $(INCDIR)
	install -m 644 $< $@

$(PKGDIR)/%.pc : %.pc
	@printf "Grabbing   %-35s > %s\n" $< $@
	mkdir -p $(PKGDIR)
	install -m 644 $< $@
```

# 生成.o目标文件及其.d依赖文件

目标`.o`文件依赖于`.cc`和`INCTATGETS`，执行命令首先编译生成`.o`文件，`-c`标志表示不需要链接；接着生成.o文件对应的依赖文件。

```makefile
$(OBJDIR)/%.o : %.cc $(INCTARGETS)
	@printf "Compiling  %-35s > %s\n" $< $@
	mkdir -p `dirname $@`
	$(CXX) -I. -I$(INCDIR) $(CXXFLAGS) -Iinclude -c $< -o $@
	@$(CXX) -I. -I$(INCDIR) $(CXXFLAGS) -Iinclude -M $< > $(@:%.o=%.d.tmp)
	@sed "0,/^.*:/s//$(subst /,\/,$@):/" $(@:%.o=%.d.tmp) > $(@:%.o=%.d)
	@sed -e 's/.*://' -e 's/\\$$//' < $(@:%.o=%.d.tmp) | fmt -1 | \
                sed -e 's/^ *//' -e 's/$$/:/' >> $(@:%.o=%.d)
	@rm -f $(@:%.o=%.d.tmp)
```

指定了`-M`选项后，gcc 会分析源代码文件及其包含的头文件，并生成一个描述文件之间依赖关系的文本文件。生成的依赖关系文件通常用于make中的自动化构建过程，它可以告诉make哪些文件需要重新编译，以及它们之间的依赖关系。生成的依赖关系文件内容类似于以下示例：

```makefile
example.o: example.c header1.h header2.h
header1.o: header1.h
header2.o: header2.h
```

其中，每一行描述一个目标文件及其依赖的头文件。这些信息可以用于构建系统中的规则，以确保在需要时重新编译相关的源代码文件。（在第七点中就有用到依赖文件）

在例子中，`gcc`首先生成的是临时依赖文件`.d.tmp`，再使用`sed`替换命令将其修改为期望的格式，最终重定向到`.d`文件中。

# 其他

最后定义了`clean`、`install`的行为，其中安装命令就是将build中的文件拷贝到系统目录下，`cp -P`表示保留链接文件的链接属性，而非拷贝实际文件的内容。`formatting.mk`则是定义了一些使用`astyle`修改源码风格的命令，`astyle`是一个开源的代码格式化工具，用于自动化地调整代码的风格和格式，它支持多种编程语言，包括C、C++、C#、Java等。

```makefile
clean :
	$(MAKE) -C device clean
	rm -rf ${INCDIR} ${LIBDIR} ${PKGDIR} ${OBJDIR}

install : build
	mkdir -p $(PREFIX)/lib
	mkdir -p $(PREFIX)/lib/pkgconfig
	mkdir -p $(PREFIX)/include
	cp -P -v $(BUILDDIR)/lib/lib* $(PREFIX)/lib/
	cp -P -v $(BUILDDIR)/lib/pkgconfig/* $(PREFIX)/lib/pkgconfig/
	cp -v $(BUILDDIR)/include/* $(PREFIX)/include/

FILESTOFORMAT := $(shell find . -name ".\#*" -prune -o \( -name "*.cc" -o -name "*.h" \) -print | grep -v -E 'ibvwrap.h|nvmlwrap.h|gdrwrap.h|nccl.h')
# Note that formatting.mk defines a new target so in order to not overwrite the default target,
# it shouldn't be included at the top. Also, it uses the above definition of FILESTOFORMAT as well
# as the BUILDDIR variable.
include ../makefiles/formatting.mk
```
