+++
title = 'IO profile 和 trace 插桩库的一些实现'
date = 2022-11-02
draft = false
toc = true
tocBorder = true
+++

## 代码思路

### 测试程序

编写两个简单的测试程序，分别调用库函数和自定义函数：

```c
[main.c]
#include <stdio.h>
#include <stdlib.h>

int main() {
  int **a = (int **)malloc(sizeof(int *) * 3);
  for (int i = 0; i < 3; ++i)
    a[i] = (int *)malloc(sizeof(int) * 5);
  return 0;
}

```

```c
[foo.c]
#include "stdio.h"

void foo(){};

void bar(int i) {
  printf("%d\n", i);
  foo();
}

int main() {
  for (int i = 0; i < 3; ++i)
    bar(i);
  return 0;
}
```

注意部分功能需使用 #define _GNU_SOURCE 启用。

### -finstrument-functions

使用 -finstrument-functions 参数，编译器会自动在函数出入口增加

```c
void __cyg_profile_func_enter (void *this_fn,
                               void *call_site);
void __cyg_profile_func_exit  (void *this_fn,
                               void *call_site);
```

从而起到注入作用。

```c
[cyg.c]
#include <dlfcn.h>
#include <stdio.h>

void __cyg_profile_func_enter(void *this_fn, void *call_site) {
  Dl_info dl_func, dl_caller;
  if (dladdr(this_fn, &dl_func) && dladdr(call_site, &dl_caller))
    printf("Entering func: %s, caller: %s\n", dl_func.dli_sname,
           dl_caller.dli_sname);
}

void __cyg_profile_func_exit(void *this_fn, void *call_site) {
  Dl_info dl_func, dl_caller;
  if (dladdr(this_fn, &dl_func) && dladdr(call_site, &dl_caller))
    printf("Exiting func: %s, caller: %s\n", dl_func.dli_sname,
           dl_caller.dli_sname);
}
```

编译使用

```bash
gcc cyg.c -c
gcc main.c cyg.o -finstrument-functions -ldl -rdynamic
```

这种方法应用于所有函数，适用于 debug 等场景，但需注意无法在库函数使用。

### -Wl,--wrap

使用 --wrap=foo 链接器会把 __real_foo 解析为 foo，并增加 __wrap_foo 供注入使用。

```c
[wrap.c]
#include <stdio.h>

void *__real_malloc(size_t size);

void *__wrap_malloc(size_t size) {
  fprintf(stderr, "malloc(%ld) = ", size);
  void *ret = __real_malloc(size);
  fprintf(stderr, "%p\n", ret);
  return ret;
}
```

编译使用

```bash
gcc main.c wrap.c -Wl,--wrap=malloc
```

### LD_PRELOAD

LD_PRELOAD 环境变量用于加载动态库，优先级最高，可以用于注入。

```c
[preload.c]
#include <dlfcn.h>
#include <stdio.h>

static void *(*real_malloc)(size_t);

static void init() {
  if ((real_malloc = dlsym(RTLD_NEXT, "malloc")) == NULL)
    fprintf(stderr, "Error in `dlsym`: %s\n", dlerror());
}

void *malloc(size_t size) {
  if (real_malloc == NULL)
    init();
  fprintf(stderr, "malloc(%ld) = ", size);
  void *ret = real_malloc(size);
  fprintf(stderr, "%p\n", ret);
  return ret;
}
```

编译使用

```shell
gcc -fPIC -shared preload.c -ldl
LD_PRELOAD=./preload.so ./main
```

这种方法无需重新编译，也无需源代码，需要函数为动态链接。

## 实验结果

### 测试程序

使用 read 与 write 编写测试程序：

```c
#include <fcntl.h>
#include <unistd.h>

int main() {
  char buf[1024];
  int fn = open("rand.txt", O_RDONLY);
  size_t nread;
  if (fn != -1) {
    while ((nread = read(fn, buf, sizeof(buf))) > 0)
      write(1, buf, nread);
    close(fn);
  }
  return 0;
}
```

调用函数详见 read(2) 与 write(2)，避免重复以 read 为例。

### LD_PRELOAD

-Wl,--wrap 和 LD_PRELOAD 本质相同，使用后者直接注入可执行文件。

共用部分

```c
#define _GNU_SOURCE

#include <dlfcn.h>
#include <stdio.h>

static ssize_t (*real_read)(int, void *, size_t);
```

支持 profile 功能

```c
static size_t read_time = 0;
static size_t read_count = 0;
static size_t read_size = 0;

void read_status() {
  fprintf(stderr, "read: %ld, %ld, %ld\n", read_time, read_count, read_size);
}

static void read_init() {
  if ((real_read = dlsym(RTLD_NEXT, "read")) == NULL)
    fprintf(stderr, "Error in `dlsym`: %s\n", dlerror());
  atexit(read_status);
}

ssize_t read(int fd, void *buf, size_t count) {
  if (real_read == NULL)
    read_init();
  clock_t tic = clock();
  ssize_t ret = real_read(fd, buf, count);
  clock_t toc = clock();
  read_time += toc - tic;
  ++read_count;
  read_size += ret;
  return ret;
}
```

支持 trace 功能

```c
static void read_init() {
  if ((real_read = dlsym(RTLD_NEXT, "read")) == NULL)
    fprintf(stderr, "Error in `dlsym`: %s\n", dlerror());
}

ssize_t read(int fd, void *buf, size_t count) {
  if (real_read == NULL)
    read_init();
  fprintf(stderr, "read(%d, %p, %ld) = ", fd, buf, count);
  ssize_t ret = real_read(fd, buf, count);
  fprintf(stderr, "%ld\n", ret);
  return ret;
}
```

可以使用 -D 等方式合并为同一程序，经测试与 iotrace 结果相同。

## 开销结果

反编译结果如下：

```asm
read(int, void*, unsigned long):
    push    rbp
    mov     rbp, rsp
    sub     rsp, 64
    mov     DWORD PTR [rbp-36], edi
    mov     QWORD PTR [rbp-48], rsi
    mov     QWORD PTR [rbp-56], rdx
    mov     rax, QWORD PTR real_read[rip]
    test    rax, rax
    jne     .L5
    call    read_init()
```

注入开销与对应函数参数有关，read 包含 9 个额外指令。

## 总结

常见注入有三种形式：

- -finstrument-functions 作用于所有用户函数
- -Wl,--wrap 在链接时注入
- LD_PRELOAD 在执行时注入，需要为动态链接函数

注入开销不大，但仍有优化空间，如 Google 工具达到了较高速度，可以参考学习。
