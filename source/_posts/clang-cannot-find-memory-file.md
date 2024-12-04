---
title: Clang++ 无法找到 <memory> 头文件
date: 2024-12-04 21:57:35 +0800
tags:
  - clang++
category: 杂项
---

最近在使用 [pg_duckdb](https://github.com/duckdb/pg_duckdb) 遇到了一个比较有趣的问题，当 PostgreSQL 启用 LLVM 时，pg_duckdb 编译失败，错误信息如下：

```
g++ -Wall -Wpointer-arith -Wendif-labels -Wmissing-format-attribute -Wimplicit-fallthrough=3 -Wcast-function-type -Wshadow=compatible-local -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -g -O2 -std=c++17 -Wno-sign-compare -Wno-register  -fPIC -fvisibility=hidden -fvisibility-inlines-hidden -Iinclude -Ithird_party/duckdb/src/include -Ithird_party/duckdb/third_party/re2 -I. -I./ -I/data/Codes/pg/REL_16_STABLE/build/pg/include/postgresql/server -I/data/Codes/pg/REL_16_STABLE/build/pg/include/postgresql/internal  -D_GNU_SOURCE -I/usr/include/libxml2   -c -o src/scan/postgres_scan.o src/scan/postgres_scan.cpp -MMD -MP -MF .deps/postgres_scan.Po
gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Wendif-labels -Wmissing-format-attribute -Wimplicit-fallthrough=3 -Wcast-function-type -Wshadow=compatible-local -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-format-truncation -Wno-stringop-truncation -g -Wmissing-prototypes -Wincompatible-pointer-types -fsanitize=bounds -fPIC -fvisibility=hidden -shared -o pg_duckdb.so  src/pgduckdb_background_worker.o src/pgduckdb.o src/pgduckdb_ddl.o src/pgduckdb_detoast.o src/pgduckdb_duckdb.o src/pgduckdb_filter.o src/pgduckdb_hooks.o src/pgduckdb_memory_allocator.o src/pgduckdb_metadata_cache.o src/pgduckdb_node.o src/pgduckdb_options.o src/pgduckdb_planner.o src/pgduckdb_ruleutils.o src/pgduckdb_table_am.o src/pgduckdb_types.o src/pgduckdb_utils.o src/catalog/pgduckdb_catalog.o src/catalog/pgduckdb_schema.o src/catalog/pgduckdb_storage.o src/catalog/pgduckdb_table.o src/catalog/pgduckdb_transaction.o src/catalog/pgduckdb_transaction_manager.o src/scan/heap_reader.o src/scan/postgres_scan.o src/scan/postgres_seq_scan.o src/utility/copy.o src/vendor/pg_explain.o  src/vendor/pg_ruleutils_15.o src/vendor/pg_ruleutils_16.o src/vendor/pg_ruleutils_17.o -L/data/Codes/pg/REL_16_STABLE/build/pg/lib   -L/usr/lib/llvm-18/lib  -Wl,--as-needed -Wl,-rpath,'/data/Codes/pg/REL_16_STABLE/build/pg/lib',--enable-new-dtags -fvisibility=hidden -Wl,-rpath,/data/Codes/pg/REL_16_STABLE/build/pg/lib/postgresql/ -lpq -Lthird_party/duckdb/build/release/src -L/data/Codes/pg/REL_16_STABLE/build/pg/lib/postgresql -lduckdb -lstdc++ -llz4
/usr/bin/clang -xc++ -Wno-ignored-attributes -fno-strict-aliasing -fwrapv -fexcess-precision=standard -O2  -Iinclude -Ithird_party/duckdb/src/include -Ithird_party/duckdb/third_party/re2 -I. -I./ -I/data/Codes/pg/REL_16_STABLE/build/pg/include/postgresql/server -I/data/Codes/pg/REL_16_STABLE/build/pg/include/postgresql/internal  -D_GNU_SOURCE -I/usr/include/libxml2  -flto=thin -emit-llvm -c -std=c++17 -Wno-sign-compare -Wno-register  -o src/pgduckdb_background_worker.bc src/pgduckdb_background_worker.cpp
In file included from src/pgduckdb_background_worker.cpp:1:
In file included from third_party/duckdb/src/include/duckdb.hpp:11:
In file included from third_party/duckdb/src/include/duckdb/main/connection.hpp:11:
In file included from third_party/duckdb/src/include/duckdb/common/enums/profiler_format.hpp:11:
third_party/duckdb/src/include/duckdb/common/constants.hpp:11:10: fatal error: 'memory' file not found
   11 | #include <memory>
      |          ^~~~~~~~
1 error generated.
make: *** [/data/Codes/pg/REL_16_STABLE/build/pg/lib/postgresql/pgxs/src/makefiles/../../src/Makefile.global:1096: src/pgduckdb_background_worker.bc] Error 1
```

`memory` 可是 [C++ 标准库中的头文件](https://cplusplus.com/reference/memory/)，居然都找不到，有点离谱了吧。

<!--more-->

在我的环境中这个问题可以稳定的重现，为此我向 pg_duckdb 提了一个 [issue](https://github.com/duckdb/pg_duckdb/issues/370)。[Yves](https://about.me/y.lemaout) 尝试通过 docker 重现这个问题，但是没有成功。我自己也尝试通过 docker 来重现，发现也不能成功复现。那么肯定是我的环境出现了问题。

我尝试使用下面的代码来测试我的环境：

```c++
#include <iostream>
#include <memory>

int
main(void)
{
    std::cout <<"Hello, world!" <<std::endl;
    return 0;
}
```

当我使用 g++ 编译时，可以正常执行。

```bash
$ g++ demo.cc -o demo-gcc
$ ./demo-gcc
Hello, world!
```

但当我使用 clang++ 编译时，遇到了下面的问题：

```bash
$ clang++ t.cc
t.cc:1:10: fatal error: 'iostream' file not found
    1 | #include <iostream>
      |          ^~~~~~~~~~
1 error generated.
$ clang -xc++ t.cc
t.cc:1:10: fatal error: 'iostream' file not found
    1 | #include <iostream>
      |          ^~~~~~~~~~
1 error generated.
```

果然是环境出了问题。通过搜索我发现 Clang 和 Clang++ 借用了 GCC 和 G++ 的头文件，如果安装了更高版本的 GCC，但没有相应的 G++，Clang++ 就会感到困惑，从而无法找到头文件。

通过运行 `clang++ -v -E` 可以查看 clang 找到了哪些 GCC 以及选择了哪个：

```bash
$ clang++ -v -E
Ubuntu clang version 18.1.3 (1ubuntu1)
Target: x86_64-pc-linux-gnu
Thread model: posix
InstalledDir: /usr/bin
Found candidate GCC installation: /usr/bin/../lib/gcc/x86_64-linux-gnu/13
Found candidate GCC installation: /usr/bin/../lib/gcc/x86_64-linux-gnu/14
Selected GCC installation: /usr/bin/../lib/gcc/x86_64-linux-gnu/14
Candidate multilib: .;@m64
Selected multilib: .;@m64
```

可以看到 clang++ 找到了 gcc-14，但是由于我没有安装 g++-14，因此出现了上述找不到头文件的问题，通过安装 g++-14 或 libstdc++-14-dev 即可解决该问题。

## 参考

[1] https://cplusplus.com/reference/memory/
[2] https://askubuntu.com/questions/1449769/clang-cannot-find-iostream
[3] https://forums.linuxmint.com/viewtopic.php?t=368329
[4] https://github.com/duckdb/pg_duckdb/issues/370

<div class="just-for-fun">
笑林广记 - 没须屁股

一公领孙溪中洗澡。孙拿得一虾，或前跳，或却走。
孙问公曰：“前赶后退，后赶前行，不知何处是头，何处是尾？”
公答曰：“有须的是头，没须的是尾。”
</div>
