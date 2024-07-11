---
title: "Livepatch selftests: the journey to Kbuild and back"
date: 2024-07-10T23:18:00-03:00
tags:
  - kbuild
  - selftests
  - livepatch
  - kernel
summary: or how moving code around ended up on a Makefile graduation course
cover:
  image: /images/understand-makefiles.jpg
---
One might think that in order to contribute to Linux kernel the developer should *speak* C and assembly. This is correct most of times, but not always.

There are cases where a simple code move between directories require changes on Kbuild, implying  knowledge of Makefile. If you have dealt with Makefiles before, you know what lies ahead of you.

# The problem
The livepatching subsystem uses kselftests to ensure that the subsystem is working as expected. The livepatch tests are placed on ``tools/testing/selftests/livepatch`` directory, and they do a very simple job: load livepatch modules and observe if they behave like they should. The source code of the livepatch testing modules live on ``lib/livepatch``  and are compiled only if **CONFIG_TEST_LIVEPATCH** is set. These test modules need to be installed like normal modules to allow the tests to work. This requirement forbids reasonable use cases:

* Running the tests on a kernel that wasn't compiled **CONFIG_TEST_LIVEPATCH**
* Changing livepatch test modules code and simply recompiling them
* Running livepatch kselftests from upstream on older kernels 

Until now, to be able to execute livepatch kselftests one should enable **CONFIG_TEST_LIVEPATCH**, build the kernel and the modules, and create a package for the test modules. This allows Linux distributions to test their kernels before releasing a new update, excluding the test modules from the final image.

The *Kernel Livepatch* team at *SUSE* wanted to change this approach to something that could be easily ported to different systems, or to different versions of the same distribution. The idea was to move the livepatch testing modules from ``lib/livepatch`` to ``tools/testing/selftests/livepatch``, compile the modules as *out-of-tree* and run the tests by loading the modules. This change would empower developers making possible to change the test modules and and simply running the tests.

The idea is pretty good, and seems very simple to implement. Just moving code from one directory into another. What could go wrong?

# Kbuild and Makefiles in the way
The Linux Kernel build system is called [Kbuild](https://docs.kernel.org/kbuild/index.html) and is composed by Makefiles. As you can image, things can get complex when dealing with multiple Makefiles that include other Makefiles. Kbuild's complexity is such that it has its own [mailing list](https://lore.kernel.org/linux-kbuild/) to discuss changes and improvements. Things are documented as much as possible, but it doesn't make the process of touching it easier.

## Problem 1: Recursive out-of-tree module building
The very first issue was found while moving the code from ``lib/livepatch`` to ``tools/testing/selftests/livepatch/test-modules``. Per the [documentation](https://docs.kernel.org/kbuild/modules.html), it should be easy to address since it's simple to build an *out-of-tree* module. Our use case was a *little* different, since our modules are *in-tree* but are being compiled as *out-of-tree*.

Following the documentation, the modules were moved from ``lib/livepatch`` to ``tools/testing/selftests/livepatch/test-modules``. The  Makefile below was created to compile them:
```make
TESTMODS_DIR := $(realpath $(dir $(abspath $(lastword $(MAKEFILE_LIST)))))
KDIR ?= /lib/modules/$(shell uname -r)/build

obj-m += test_klp_atomic_replace.o \
	test_klp_callbacks_busy.o \
	test_klp_callbacks_demo.o \
	test_klp_callbacks_demo2.o \
	test_klp_callbacks_mod.o \
	test_klp_livepatch.o \
	test_klp_state.o \
	test_klp_state2.o \
	test_klp_state3.o \
	test_klp_shadow_vars.o

modules:
	$(Q)$(MAKE) -C $(KDIR) modules M=$(TESTMODS_DIR)
clean:
	$(Q)$(MAKE) -C $(KDIR) clean M=$(TESTMODS_DIR)  
```

The file ``tools/testing/selftests/livepatch/Makefile`` was changed to set **TEST_GEN_MODS_DIR := test_modules** in order to enable building the modules that were moved into ``test_modules`` directory before running the tests that will load these modules.

With all preparations finished, it's time to start kseltest to build and run the tests, but the build already failed:
```sh
make kselftest KDIR=$(pwd) TARGETS=livepatch
  CC      scripts/mod/empty.o
  MKELF   scripts/mod/elfconfig.h
  HOSTCC  scripts/mod/modpost.o
  CC      scripts/mod/devicetable-offsets.s
  HOSTCC  scripts/mod/file2alias.o
  HOSTCC  scripts/mod/sumversion.o
  HOSTCC  scripts/mod/symsearch.o
  HOSTLD  scripts/mod/modpost
m2c    -o scripts/Makefile.build.o scripts/Makefile.build.mod
make[6]: m2c: No such file or directory
```

What? Per the [Make documentation](https://www.gnu.org/software/make/manual/html_node/Implicit-Variables.html#index-M2C), m2c is the Modula to C compiler. What's going on?  I sent a message to [Kbuild mailing list](https://lore.kernel.org/linux-kbuild/lp2gjgzwxvhluh7fpmmo2drhii7bxcrlvxacclfgsl4ycubjhc@jjq2jfvow4y2/) exposing my problem and even created a small reproducer of the problem. The Kbuild maintainer suggested changing **M=** to **KBUILD_EXTMOD=**, which *solved* the issue!

The only time the issue was being observed was when the build process started for the toplevel directory of the Linux Kernel. It made me look into the toplevel Makefile's [code](https://elixir.bootlin.com/linux/v6.9.1/source/Makefile#L141) discovering the problem: the **M=** argument is only parsed once for recursive Makefile rules, but it would work fine if the tests were executed like below:

```sh
make -C tools/testing/selftests/livepatch run_tests
```

## Makefiles: from top to livepatch selftests

### Toplevel Makefile
In order to understand problem, we should first understand the flow that Kbuild takes to build the modules and run the tests. The traverse starts from the toplevel Makefile when we execute the make invocation below:

```sh
make kselftest KDIR=$(pwd) TARGETS=livepatch
```

On the toplevel Makefile some specific variables as checked, like **M=** or **V=**, but in this case only **KDIR** and **TARGETS** is being passed, and these don't affect the build procedure at this point. Before reaching the **kselftest** target it sets a variable called **sub_make_done**. *This part is very important! Remember the name of this variable!* 

The **kselftest** target will execute the command below, jumping into ``tools/testing/selftests`` directory:

```sh
$(Q)$(MAKE) -C $(srctree)/tools/testing/selftests run_tests
```

### tools/testing/selftests/Makefile 

This Makefile checks the *TARGETS* variable that was propagated from the toplevel Makefile, and then jumps into the directory specified by *TARGETS* to run the *run_tests* target:
```make
run_tests: all                                                                                                        
        @for TARGET in $(TARGETS); do \                                                                               
                BUILD_TARGET=$$BUILD/$$TARGET;  \                                                                     
                $(MAKE) OUTPUT=$$BUILD_TARGET -C $$TARGET run_tests \                                                 
                                SRC_PATH=$(shell readlink -e $$(pwd)) \                                               
                                OBJ_PATH=$(BUILD)                   \                                                 
                                O=$(abs_objtree);                   \                                                 
        done; 
```

### tools/testing/selftests/livepatch/Makefile
This Makefile will set variables used by kselftests:
```make
TEST_GEN_FILES := test_klp-call_getpid                                                                                
TEST_GEN_MODS_DIR := test_modules                                                                                     
TEST_PROGS_EXTENDED := functions.sh                                                                                   
TEST_PROGS := \                                                                                                       
        test-livepatch.sh \                                                                                           
        test-callbacks.sh \                                                                                           
        test-shadow-vars.sh \                                                                                         
        test-state.sh \                                                                                               
        test-ftrace.sh \                                                                                              
        test-sysfs.sh \                                                                                               
        test-syscall.sh                                                                                               
                                                                                                                      
TEST_FILES := settings                                                                                                
                                                                                                                      
include ../lib.mk
```

The most important variable now is the **TEST_GEN_MODS_DIR**, which specifies the directory containing modules to be built. Also important is the inclusion of another makefile: ``lib.mk``. This file contains general targets to run selftests, and uses the variables set above to work properly. Let's see what ``lib.mk`` does when *TEST_GEN_MODS_DIR* is specified:

```make
...
all: $(TEST_GEN_PROGS) $(TEST_GEN_PROGS_EXTENDED) $(TEST_GEN_FILES) \                                                 
        $(if $(TEST_GEN_MODS_DIR),gen_mods_dir)
...

run_tests: all         
ifdef building_out_of_srctree                    
    @if [ "X$(TEST_PROGS)$(TEST_PROGS_EXTENDED)$(TEST_FILES)$(TEST_GEN_MODS_DIR)" != "X" ]; then \            
                rsync -aq --copy-unsafe-links $(TEST_PROGS) $(TEST_PROGS_EXTENDED) $(TEST_FILES) $(TEST_GEN_MODS_DIR) $(OUTPUT); \  
    fi
	@$(INSTALL_INCLUDES)
    @if [ "X$(TEST_PROGS)" != "X" ]; then \                
		$(call RUN_TESTS, $(TEST_GEN_PROGS) $(TEST_CUSTOM_PROGS) \
							$(addprefix $(OUTPUT)/,$(TEST_PROGS))) ; \
    else \
		$(call RUN_TESTS, $(TEST_GEN_PROGS) $(TEST_CUSTOM_PROGS)); \                                                                
    fi                
else                                                                                                                               
	@$(call RUN_TESTS, $(TEST_GEN_PROGS) $(TEST_CUSTOM_PROGS) $(TEST_PROGS))                
endif

gen_mods_dir:                                                                                                         
        $(Q)$(MAKE) -C $(TEST_GEN_MODS_DIR)                                                                           
                                                                                                                      
clean_mods_dir:                                                                                                       
        $(Q)$(MAKE) -C $(TEST_GEN_MODS_DIR) clean
```

The *run_tests* depends on *all* target, which check some variables. If *TEST_GEN_MODS_DIR* is set ``lib.mk`` executed the target ``gen_mods_dir``, which will jump into the directory specified by it to build the modules, before running the tests specified by **TEST_PROGS**.

### tools/testing/selftests/livepatch/test_modules/Makefile
This Makefile (which was already shown before) is the responsible 

```make
TESTMODS_DIR := $(realpath $(dir $(abspath $(lastword $(MAKEFILE_LIST)))))
KDIR ?= /lib/modules/$(shell uname -r)/build

obj-m += test_klp_atomic_replace.o \
	test_klp_callbacks_busy.o \
	test_klp_callbacks_demo.o \
	test_klp_callbacks_demo2.o \
	test_klp_callbacks_mod.o \
	test_klp_livepatch.o \
	test_klp_state.o \
	test_klp_state2.o \
	test_klp_state3.o \
	test_klp_shadow_vars.o

modules:
	$(Q)$(MAKE) -C $(KDIR) modules M=$(TESTMODS_DIR)
clean:
	$(Q)$(MAKE) -C $(KDIR) clean M=$(TESTMODS_DIR) 
```

The code above follow the rule of building an *out-of-tree* module: it contains a couple of *.o* files that will result in different modules, a **KDIR** to jump into, and uses the modules target, along with the **M=** path being set to the current directory. At this point the **KDIR** already contains the value we passed when we started the process, pointing to the toplevel directory. Make will jump into **KDIR**.

### Toplevel Makefile (again)
We are back into the toplevel directory, but now with **M=** being set, meaning that we are about to build the modules pointed by it. Everything seems fine, but the error message appears:

```sh
...
  CC      scripts/mod/empty.o
  MKELF   scripts/mod/elfconfig.h
  HOSTCC  scripts/mod/modpost.o
  CC      scripts/mod/devicetable-offsets.s
  HOSTCC  scripts/mod/file2alias.o
  HOSTCC  scripts/mod/sumversion.o
  HOSTCC  scripts/mod/symsearch.o
  HOSTLD  scripts/mod/modpost
m2c    -o scripts/Makefile.build.o scripts/Makefile.build.mod
make[6]: m2c: No such file or directory
```

If we inspect the toplevel Makefile closer, the process of checking if **M=** was set is done only if **sub_make_done** isn't set, meaning we only check **M=** on the first invocation of the recursive make calls.

In the end, the Kbuild maintainer [suggested](https://lore.kernel.org/linux-kbuild/CAK7LNATLv2KSWo0BnFGXi73GVdnvc1EX23TvTkKT1U-krgBnNQ@mail.gmail.com/) me to use **KBUILD_EXTMOD=**  instead of **M=** and that did the trick! The entire flow of the recursive Makefile invocations can be see below:

```
Makefile (toplevel)
tools/testing/selftests/Makefile
tools/testing/selftests/livepatch/Makefile
tools/testing/selftests/livepatch/test-modules/Makefile
Makefile (toplevel)
```

The value of **M=** is used to set **KBUILD_EXTMOD=** either way, so using the later solved the problem, the modules are built and the tests were run as expected:

```sh
$  sudo make kselftest KDIR=$(pwd) TARGETS=livepatch
  CC       test_klp-call_getpid
  CC [M]  /home/mpdesouza/git/linux/tools/testing/selftests/livepatch/test_modules/test_klp_atomic_replace.o
  CC [M]  /home/mpdesouza/git/linux/tools/testing/selftests/livepatch/test_modules/test_klp_callbacks_busy.o
  CC [M]  /home/mpdesouza/git/linux/tools/testing/selftests/livepatch/test_modules/test_klp_callbacks_demo.o
  ...
  MODPOST /home/mpdesouza/git/linux/tools/testing/selftests/livepatch/test_modules/Module.symvers
  CC [M]  /home/mpdesouza/git/linux/tools/testing/selftests/livepatch/test_modules/test_klp_atomic_replace.mod.o
  LD [M]  /home/mpdesouza/git/linux/tools/testing/selftests/livepatch/test_modules/test_klp_atomic_replace.tmp.ko
  KLP     /home/mpdesouza/git/linux/tools/testing/selftests/livepatch/test_modules/test_klp_atomic_replace.ko
  CC [M]  /home/mpdesouza/git/linux/tools/testing/selftests/livepatch/test_modules/test_klp_callbacks_busy.mod.o
  LD [M]  /home/mpdesouza/git/linux/tools/testing/selftests/livepatch/test_modules/test_klp_callbacks_busy.ko
  BTF [M] /home/mpdesouza/git/linux/tools/testing/selftests/livepatch/test_modules/test_klp_callbacks_busy.ko
  CC [M]  /home/mpdesouza/git/linux/tools/testing/selftests/livepatch/test_modules/test_klp_callbacks_demo.mod.o
  LD [M]  /home/mpdesouza/git/linux/tools/testing/selftests/livepatch/test_modules/test_klp_callbacks_demo.tmp.ko
  KLP     /home/mpdesouza/git/linux/tools/testing/selftests/livepatch/test_modules/test_klp_callbacks_demo.ko
  ...
TAP version 13
1..7
# timeout set to 0
# selftests: livepatch: test-livepatch.sh
# TEST: basic function patching ... ok
# TEST: multiple livepatches ... ok
# TEST: atomic replace livepatch ... ok
ok 1 selftests: livepatch: test-livepatch.sh
...
# timeout set to 0
# selftests: livepatch: test-syscall.sh
# TEST: patch getpid syscall while being heavily hammered ... ok
ok 7 selftests: livepatch: test-syscall.sh
```

## Problem 2: Target recipes
After the patches got [merged](https://lore.kernel.org/linux-kselftest/fc9ccbf3-6b7d-46d3-a8da-54d5dc78a81c@linuxfoundation.org/#t), I sent another round of patches to simplify the handling of *TEST_GEN_MODS_DIR*, specially [this one](https://lore.kernel.org/linux-kselftest/20240221-lp-selftests-fixes-v2-4-a19be1e029a7@suse.com/#t). The patch introduced a *warning* because of the *all* target on ``lib.mk`` (that is included by all selftests Makefiles), but somehow started to fail only with this patch alone. Fortunately a person from the mailing list reported the [issue](https://lore.kernel.org/linux-kselftest/ZdgTkKSSme5Evgwq@yujie-X299/), making me scratch my head to discover what was wrong. Well, let's check how Makefiles work with duplicated targets.

After tons of experiments I found that you can have duplicated Makefile targets only if they don't contain a recipe on each target. Otherwise a warning will be shown, and only the recipe of the last target will be executed, but the dependencies of both targets will be evaluated. Take for example the files below:

```
all: dep
	@echo "override.mk: running all target"
dep:
	@echo "override.mk: running dep target"
	
.PHONY: all dep  
```

Let's call this file ``override.mk``. And now let's look into another file on the same directory called ``main.mk``:
```
include override.mk

all: local_dep                                              
	@echo "main.mk: running recipe of all target"                                          

local_dep:
	@echo "main.mk: running local_dep target"

.PHONY: all local_dep 
```

You can run them by:
```
make -f main.mk
```

And the result would be:
```
main.mk:4: warning: overriding recipe for target 'all'
override.mk:2: warning: ignoring old recipe for target 'all'
main.mk: running local_dep target
override.mk: running dep target
main.mk: running recipe of all target
```

As you can see, the dependencies all both *all* targets were evaluated/executed, but only the recipe of *all* target from ``main.mk`` executed. At this point, I asked the maintainer of kselftests to ignore that specific patch while the other cleanup patches of the series were accepted.
## Closing thoughts

After the changes landed, I discussed with other kernel developers about the journey, and they agreed that touching Kbuild/Makefiles is challenging. I even created a [git repository](https://github.com/marcosps/makefile-examples) to exercise Makefile constructions that I faced on Kbuild to understand better how they behave. You might find this useful.

If you feel frustrated trying to understand Kbuild, consider that kernel developers in general avoid touching Kbuild, specially because they don't understand all the intricate details of it.

Thanks for reading!
# References
[Official Kbuild documentation](https://docs.kernel.org/kbuild/index.html)

[Kbuild modules documentation](https://docs.kernel.org/kbuild/modules.html)

[My Makefile examples repository](https://github.com/marcosps/makefile-examples)
