# UNIX基础

fd是文件描述符（file descriptor），内核用以标识一个特定进程正在访问的文件。当内核打开一个现有文件或创建一个新文件时，它都返回一个文件描述符。

两个常量STDIN_FILENO和STDOUT_FILENO定义在<unistd.h>头文件中，它们制定了标准输入和标准输出的文件描述符。

# 文件IO

## 3.3 函数open和openat

```c
#include <fcntl.h>
int open(const char *path, int oflag, ..);
int openat(int fd, const char *path, int oflag, ...);
```

path是文件名字，多个参数可以用`|`运算构成oflag参数。

## 3.4 函数creat

```c
#include <fcntl.h>
int creat(const char *path, mode_t mode);
```

等效于

```c
open(path, O_WRONLY|O_CREAT|O_TRUNC, mode);
```

creat只能以WRONLY方式打开所创建的文件，所以有必要使用open替代。

## 3.6 函数lseek

```c
#include <unistd.h>
off_t lseek(int fd, off_t offset, int whence);
```

对offset的解释与whence有关

- `SEEK_SET`，距文件开始处offset个字节
- `SEEK_CUR`，当前值加offset
- `SEEK_END`，文件长度加offset

文件偏移量可以大于文件当前长度。这种情况下，对文件的下一次写将加长该文件并形成空洞（hole）。文件中没有写过的byte被读为0。新写的数据需要分配磁盘块，但文件空洞不占用磁盘块。

## 3.7 函数read

```c
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t, nbytes);
```

## 3.8 函数write

```c
#include <unistd.h>
ssize_t write(int fd, const void *buf, size_t nbytes);
```

## 3.9 IO效率

大多数文件系统都采用某种预读（read ahead）技术，当检测到正进行顺序读写，系统就试图读入比Application要求更多的数据，并假想Application很快就会读这些数据。

## 3.10 文件共享

1. Every process has an entry in the process table. Within each process table entry is a table of open file descriptors, which we can think of as a vector, with one entry per descriptor. 每个进程在进程表中都有一个记录项，记录项中包含一张打开文件描述符表，可将其视为一个矢量，每个描述符占用一项。
   1. 文件描述符标志
   2. 指向一个文件表项的指针
2. The kernel maintains a file table for all open files.
   1. 文件状态标志
   2. 当前文件偏移量
   3. 指向该文件v节点表项的指针
3. 每个打开文件都有一个v节点。v节点包含了文件类型和对此文件进行各种操作函数的指针，以及包含了i节点。i节点在4.14节说明。

**该进程有两个不同的打开文件：**

![](E:\EBOOK\GITnote\APUE_3rd\APUE_pics\image-20210626235853633.png)



**两个独立进程各自打开同一文件：**

![](E:\EBOOK\GITnote\APUE_3rd\APUE_pics\image-20210626235923510.png)

## 3.11 Atomic Operations

原子操作指的是由多步组成的一个操作，要么执行完所有步骤，要么一步也不执行，不可能只执行步骤的一个子集。

### Appending to a File

在打开文件时设置0_APPEND标志，这样使得内核在每次写之前，都将进程的当前偏移量设置到该文件尾端处。

### 函数pread和pwrite

调用pread相当于调用lseek后调用read。调用pread时，无法中断其定位和读操作，也不更新当前文件偏移量。

## 3.12 函数dup和dup2

```c
#include <unistd.h>
int dup(int oldfd);
int dup2(int oldfd, int newfd);
```

当调用dup函数时，内核在进程中创建一个新的文件描述符，此描述符是当前可用文件描述符的最小数值，这个文件描述符指向oldfd所拥有的文件表项。

dup2和dup的区别就是可以用newfd参数指定新描述符的数值，如果newfd已经打开，则先将其关闭。如果newfd等于oldfd，则dup2返回newfd, 而不关闭它。dup2函数返回的新文件描述符同样与参数oldfd共享同一文件表项。

## 3.13 函数sync、fsync和fdatasync

```c
#include <unistd.h>
int fsync(int fd);
int fdatasync(int fd);
```

fsync函数只对由文件描述符fd指定的一个文件起作用，并且等待写磁盘操作结束才返回。fsync会同步更新文件属性，fdatasync只影响文件的数据部分。

## 3.14 函数fcntl

```c
#include <fcntl.h>
int fcntl(int fd, int cmd, ...);
```

fcntl有以下5种功能：

1. 复制一个已有的描述符。`cmd = F_DUPFD 和 F_DUPFD_CLOEXEC`
2. 获取/设置文件描述符标志。`cmd = F_GETFD 和 F_SETFD`
3. 获取/设置文件状态标志。`cmd = F_SETFL 和 F_SETFL`
4. 获取/设置异步I/O所有权。`cmd = F_GETDOWN 和 F_SETDOWN`
5. 获取/设置记录锁。`cmd = F_GETLK、F_SETLK 和 F_SETLKN`

### 习题3.2

```c
#include <apue.h>

int my_dup2(int fd, int newfd){
    if(fd==newfd) return newfd;
	if(fd<0||fd>FOPEN_MAX){
        printf("fd is wrong.\n");
        return -1;
    }
    if(newfd <0||newfd>FOPEN_MAX){
        printf("newfd is wrong.\n");
        return -1;
    }
	close(newfd);
	int fileindex[newfd+1];
	int index=0;
	while((fileindex[index++]=dup(fd))!=newfd){
		printf("result after dup(fd):%d\n",fileindex[index-1]);
		if(fileindex[index-1]==-1){
			err_sys("my_dup error!");
			return -1;
		}
	}
	int i=0;
	for(;i<index-1;i++){
		close(fileindex[i]);
	}
	return fileindex[index-1];
}
```



# 文件和目录

## 4.2 函数stat、fstat、fstatat和lstat

```c
#include <sys/stat.h>
int stat(const char *restrict pathname, struct stat *restrict buf);
int fstat(int fd, struct stat *buf);
int lstat(const char *restrict pathname, struct stat *restrict buf);
int fstatat(int fd, const char *restrict pathname, struct stat *restrict buf, int flag);
```

stat函数返回与此pathname文件有关的信息结构。使用stat函数最多的地方可能就是`ls -l`命令。

fstat函数获得已在fd上打开文件的有关信息。

lstat类似于stat，但是当命名文件是一个符号链接时，lstat返回该符号链接的有关信息，而不是该符号链接引用的文件的信息。

fstatat为一个相对于当前打开目录（由fd指向）的路径名返回文件统计信息，flag参数控制着是否跟随一个符号链接。

buf是一个指针，指向一个我们必须提供的结构，虽然具体实现不同，但有基本形式。

## 4.3 文件类型

下表在stat结构的`st_mode`成员中

|     宏     | 文件类型                            |
| :--------: | :---------------------------------- |
| S_ISREG()  | regular file 普通文件               |
| S_ISDIR()  | directory file 目录文件             |
| S_ISCHR()  | character special file 字符特殊文件 |
| S_ISBLK()  | block special file 块特殊文件       |
| S_ISFIFO() | pipe or FIFO 管道或FIFO             |
| S_ISLNK()  | symbolic link 符号链接              |
| S_ISSOCK() | socket 套接字                       |

## 4.5 文件访问权限

进程每次打开、创建或删除一个文件时，内核就进行文件访问权限测试，这种测试涉及

- 文件的所有者（`st_uid`和`st_gid`）
- 进程的有效ID（有效用户ID和有效组ID）
- （若支持的话）进程的附属组ID

两个所有者ID是文件的性质，而两个有效ID和附属组ID是进程的性质。

内核进行的测试具体如下：

1. 若进程的有效用户是0（the superuser），则允许访问。
2. 若进程的有效用户ID等于文件所有者ID（也就是进程拥有此文件），那么如果所有者<u>适当的访问权限位</u>被设置，则允许访问，否则拒绝。
3. 若进程的有效组ID或进程的附属组ID之一等于文件的组ID，那么如果组<u>适当的访问权限位</u>被设置，则允许访问，否则拒绝。
4. 若其他用户<u>适当的访问权限位</u>被设置，则允许访问，否则拒绝。

一旦进入到某一步骤内进行判断，后边的步骤不再执行。

## 4.7 函数access和faccessat

```c
#include <unistd.h>
int access(const char *pathname, int mode);
int faccessat(int fd, const char *pathname, int mode, int flag);
```

该函数为了按照实际用户ID和实际组ID进行访问权限测试，该测试与**4.5节**所述一样，但将**有效**改为**实际**。

## 4.8 函数umask

```c
#include <sys/stat.h>
mode_t mask(mode_t cmask);
```

cmask是由**4.5节**的9个访问权限位的若干个**`按位或`**构成的。

在进程创建一个新文文件或新目录时，一定会用文件模式创建屏蔽字，在文件模式创建屏蔽字中为1的位，在文件mode中的相应位一定被关闭。

## 4.9 函数chmod、fchmod和fchmodat

```c
#include <sys/stat.h>
int chmod(const char *pathname, mode_t mode);
int fchmod(int fd, mode_t mode);
int fchmodat(int fd, const char *pathname, mode_t mode, int flag);
```

该函数使我们可以更改现有文件的访问权限。chmod在指定的文件上进行操作，fchmod对已打开的文件进行操作。

fchmodat与chmod在以下两种情况中是相同的：

- pathname参数是绝对路径
- fd取值`AT_FDCWD`，pathname参数为相对路径

否则，fchmodat计算相对于fd指向目录的pathname。flag参数设置`AT_SYMLINK_NOFOLLOW`,fchmodat不会跟随符号链接。

为了改变一个文件的权限位，进程的有效用户ID必须等于文件的所有者ID，或者该进程必须具有超级用户权限。

参数mode是由9个文件访问权限位，和两个设置常量（`S_ISUID`和`S_ISGID`）、保存正文常量（`S_ISVTX`），3个组合常量（`S_IRWXU`、`S_IRWXG`和`S_IRWXO`）**`按位或`**组成的。

## 4.11 函数chown、fchown、fchownat和lchown

```c
#include <unistd.h>
int chown(const char *pathname, uid_t owner, gid_t group);
int fchown(int fd, uid_t owner, gid_t group);
int fchownat(int fd, const char *pathname, uid_t owner, gid_t group, int flag);
int lchown(const char *pathname, uid_t owner, gid_t group);
```

该函数可用于更改文件的用户ID和组ID。

## 4.14 文件系统

![](E:\EBOOK\GITnote\APUE_3rd\APUE_pics\image-20210629223810473.png)

- stat结构中，链接计数包含在st_nlink成员中，只有链接计数减少到0时，才可删除该文件。这种链接是硬链接。
- 另一种链接类型称为符号链接，符号链接文件的实际内容包含了该符号链接指向的文件的名字。比如：`lrwxrwxrwx 1 root 7 Sep 25 07:14 lib -> usr/lib` 目录项中的文件名是3个字符的字符串`lib`，该文件包含了7字节的数据`usr/lib`。
- stat结构大多数信息都取自i节点，只有**文件名**和**i节点编号**存放在目录项中。
- `mv`命令的实质：将文件`/usr/lib/foo`重命名为`/usr/foo`，如果目录`/usr/lib`和`/usr`在同一文件系统中，则文件`foo`无需移动，只需构造一个指向现有i节点的新目录项并删除旧目录项。



![](E:\EBOOK\GITnote\APUE_3rd\APUE_pics\image-20210629224212031.png)

## 4.15 函数link、linkat、unlink、unlinkat和move

```c
#include <unistd.h>
int link(const char *pathname, const char *newpath);
int linkat(int efd, const char *existingpath, int nfd, const char *newpath, int flag);
```

link函数创建一个新目录项，增加链接计数，引用现有文件`existingpath`。调用unlink函数删除一个现有的目录项，并将由pathname引用的文件的链接计数`-1`。

```c
#include <unistd.h>
int unlink(const char *pathname);
int unlinkat(int fd, const char *pathname, int flag);
```

remove函数可以解除对一个文件或目录的链接。对文件，`remove = unlink`；对目录，`remove = rmdir`。

```c
#include <stdio.h>
int remove(const char *pathname);
```

### 习题4.6

```c
#include "apue.h"
#include "myerr.h"
#include <unistd.h>
#include <fcntl.h>

int main(int argc, char *argv[])
{   
    if(argc != 3)
        err_sys("usage: ./EX_4_6.o <src path> <dst path>"); 
    char *src_path = argv[1];
    char *dst_path = argv[2];
    int fd = open(src_path, O_RDWR);
    int fd2 = open(dst_path, O_WRONLY); 
    int n;
    char buf[100] = "1234567890123456789";
    char buf2;  

    /* 给源文件制造空洞，结束后将源文件偏移量设为0 */
    int offset = lseek(fd, 100, SEEK_SET);
    printf("offset of src is %d\n", offset);
    printf("the buf is %s\n", buf);
    if(write(fd, buf, 100) < 0)
        err_sys("write to source file error.");
    lseek(fd, 0, SEEK_SET);

    /* 将源文件内容复制到目标文件 */
    while((n = read(fd, &buf2, 1)) == 1)
    {
        /* 如果读到'\0'，则不复制 */
        if(buf2 == '\0')
            continue;
        if(write(fd2, &buf2, 1) != 1)
            err_sys("write error.");
    }
    if(n < 0)
        err_sys("read error.");
    close(fd);
    close(fd2);
    exit(0);
}
```

# 标准I/O库

## 5.2 流和FILE对象

流的定向决定了所读所写的字符是单字节还是多字节（宽）字符集。mode参数为正，则指定流宽定向；mode参数为负，则指定流字节定向。

```c
#include <stdio.h>
#include <wchar.h>
int fwide(FILE *fp, int mode);
```

## 5.4 缓冲

标准I/O提供了三种类型的缓冲。

- 全缓冲：在一个流上执行第一次IO操作时，调用malloc获得缓冲区，在内容填满标准IO缓冲区后才进行实际IO操作。
- 行缓冲：流涉及终端时，通常使用行缓冲，在输入输出中遇到换行符时才执行IO操作。
- 不带缓冲：标准错误流使得错误信息尽快显示出来，所以不带缓冲。

冲洗（flush）在标准IO库方面，意味着将缓冲区中的内容写到磁盘上。在终端驱动程序方面，flush（刷清）表示丢弃已存储在缓冲区中的数据。

```c
#include <stdio.h>
void setbuf(FILE *restrict fp, char *restrict buf);
int setvbuf(FILE *restrict fp, char *restrict buf, int mode, stze_t size);
```

参数buf指向一个缓冲区，则setbuf打开缓冲机制；指向NULL则关闭缓冲。

setvbuf通过mode参数精确说明缓冲类型。

任何时候我们都可以强制冲洗一个流。fflush使该流所有未写的数据都被传送至内核。

```c
#include <stdio.h>
int fflush(FILE *fp);
```

## 5.5 打开流

```c
#include <stdio.h>
FILE *fopen(const char *restrict pathname, const char *restrict type);
FILE *freopen(const char *restrict pathname, const char *restrict type, FILE *restrict fp);
FILE *fdopen(int fd, const char *type);
```

fopen用于打开路径名为pathname的一个指定文件。

freopen一般用于将一个制定的文件打开为一个预定义的流：标准输入、标准输出或标准错误。

fdopen取一个已有的fd，并使一个标准的IO流与该描述符相结合，常用于由<u>创建管道和网络通信通道函数返回的</u>描述符。

## 5.6 读和写流

```c
#include <stdio.h>
int getc(FILE *fp);
int fgetc(FILE *fp);
int getchar(void);
/*若成功，返回下一个字符；失败eof*/
int ferror(FILE *fp);
int feof(FILE *fp);
/*上面三个函数返回相同的值（对于错误或到达文件尾），判断是哪一种为真*/
void clearerr(FILE *fp);
/*清除错误标志*/
int ungetc(int c, FILE *fp);
int putc(int c, FILE *fp);
int fputc(int c, FILE *fp);
int putchar(int c);
/*若成功，返回c；失败eof*/
```

函数getchar等于getc（stdin）。getc和fgetc的区别：getc可被实现为宏，fgetc是一个函数，不能实现为宏。

## 5.12 实现细节

每个标准IO流都有一个与其相关联的文件描述符，可以对一个流调用fileno函数以获得其描述符。

```c
#include <stdio.h>
int fileno(FILE *fp);
```

### 习题5.1

```c
void my_setbuf(FILE *restrict fp, char *restrict buf)
{
    setvbuf(fp, buf, buf? _IOFBF: _IONBF, BUFSIZ);
}
```

### 习题5.5

```c
#include "apue.h"

int fsync_streamaa(FILE *);
int main(void)
{
    FILE *fp;
    if ((fp = fopen("txt.txt", "w+")) == NULL)
    {
        perror("fopen error!");
        exit(-1);
    }
    fprintf(fp, "As I lay dying");
    fsync_streamaa(fp);
    _Exit(0);
}
int fsync_streamaa(FILE *fp)
{
    fflush(fp);
    return (fsync(fileno(fp)));
}
```

# 进程环境

## 7.3 进程终止

![](E:\EBOOK\GITnote\APUE_3rd\APUE_pics\image-20210702123445873.png)

有8种方式终止进程。5种正常终止：从main返回，调用exit，调用_exit或 _Exit，最后一个线程从其启动例程返回，从最后一个线程调用pthread_exit。3种异常终止：调用abort，接到一个信号最后一个线程对取消请求做出响应。

## 7.5 环境表

![](E:\EBOOK\GITnote\APUE_3rd\APUE_pics\image-20210702124225052.png)

每个程序都接收到一张环境表。在历史上，大多UNIX系统支持main带3个参数，第三个参数就是环境表地址：`int main(int argc, char *argv[], char *envp[]);`，后来ISO C规定了main只有两个参数。

## 7.6 C程序的存储空间布局

![](E:\EBOOK\GITnote\APUE_3rd\APUE_pics\image-20210702124752539.png)

C程序由下列几部分组成：

- 正文段，由CPU执行的机器指令部分。
- 初始化数据段，通常称数据段，包含了程序中需明确赋初值的变量。
- 未初始化数据段，通常称bss段，该名称来自于早期汇编的一个操作符`“block started by symbol”`，在程序开始执行之前，内核将数据初始化为0或空指针。
- Heap, where dynamic memory allocation usually takes place. Historically, the heap has been located between the uninitialized data and the stack. 
- Stack, where automatic variables are stored, along with information that is saved each time a function is called. Each time a function is called, the address of where to return to and certain information about the caller’s environment, are saved on the stack. The newly called function then allocates room on the stack for its automatic and temporary variables. This is how recursive functions in C can work. Each time a recursive function calls itself, a new stack frame is used, so one set of variables doesn’t interfere with the variables from another instance of the function. 

## 7.10 函数setjmp和longjmp

在c语言中，goto语句是不能跨越函数的，而执行这种类型跳转功能的是函数setjmp和longjmp，这两个函数对于处理发生在深层嵌套函数调用的出错情况是非常有用的。出错后不需要以检查错误返回值的方法逐层返回，而是在栈上跳过若干调用帧，返回到当前函数调用路径上的某一个函数中。

```c
#include <setjmp.h>
int setjmp(jmp_buf env);
void longjmp(jmp_buf env, int val);
```

把setjmp设置在程序开头，用来处理错误情况，在深层函数中以setjmp的env参数和一个非零val调用longjmp，longjmp就会将val返回到setjmp处。一个setjmp可以对应多个longjmp，这可以用val的不同值来区分。 

跳转之后自动变量、寄存器变量大多数实现不回滚，但标准称他们的值是不确定的。易失变量（volatile）、全局变量和静态变量都保持不变。

## 7.11 函数getrlimit和setrlimit

每个进程都有一组资源限制，其中一些可以用这两个函数查询和修改。

```c
#include <sys/resource.h>
int getrlimit(int resource, struct rlimit *rlptr);
int setrlimit(int resource, const struct rlimit *rlptr);
```

对函数的每次调用都指定一个资源以及一个指向下列结构的指针。

```c
struct rlimit {
    rlim_t rlim_cur; /* soft limit: current limit */
    rlim_t rlim_max; /* hard limit: maximum value for rlim_cur */
};
```



# 进程控制

## 8.3 函数fork

```c
#include <unistd.h>
pid_t fork(void);
```

一个现有的进程可以调用fork函数创建一个新进程。由fork创建的新进程称为子进程（child process）。fork函数被调用一次，返回两次。子进程中返回值是0，父进程的返回值是新建子进程的进程ID。

![](E:\EBOOK\GITnote\APUE_3rd\APUE_pics\image-20210702150927043.png)

在fork之后处理fd有两种常见的方式：

- 父进程等待子进程完成。
- 父进程和子进程各自执行不同的程序段。这种情况下，父进程和子进程各自关闭他们不需要使用的fd，就不会干扰对方使用的fd。这种方法是网络服务进程经常使用的。

父进程和子进程的区别：

- fork返回值不同。
- 进程ID不同。
- 子进程的父进程ID是创建它的进程ID；父进程的父进程ID不变。
- 子进程的`tms_utime`、`tms_stime`、`tms_cutime`、`tms_ustime`值设置为0。
- 子进程不继承父进程设置的文件锁。
- 子进程的未处理闹钟被清除。
- 子进程的未处理信号集设置为空集。 

## 8.5 函数exit

一个已经终止、但是其父进程尚未对其进行善后处理（获取终止子进程的有关信息、释放它仍占用的资源）的进程称为僵死进程（Zombie）。

## 8.6 函数wait和waitpid

当一个进程正常或异常终止时，内核就向其父进程发送SIGCHLD信号。如果进程由于接收到SIGCHLD信号而调用wait，我们期望wait会立即返回，但如果在任意时间调用wait则进程可能会阻塞。

```c
#include <sys/wait.h>
pid_t wait(int *statloc);
pid_t waitpid(pid_t pid, int *statloc, int options);
```

在一个子进程终止前，wait使其调用者阻塞，而waitpid有一选项可使调用者不阻塞。waitpid并不等待其调用后的第一个终止子进程，它有若干个选项可以控制它所有等待的进程。

参数statloc是一个整形指针，终止进程的终止状态存放在它所指的单元内，如果不关心终止状态就定义为空指针。

**两次fork：**

![](E:\EBOOK\GITnote\APUE_3rd\APUE_pics\image-20210707214903110.png)

如上图3所示，为了避免子进程child成为僵尸进程，我们可以人为地创建一个子进程child1，再让child1成为工作子进程child2的父进程，child2出生后child1退出，这个时候child2相当于是child1产生的孤儿进程，这个孤儿进程由系统进程init回收。这样，当child2退出的时候，init就会回收child2的资源，child2就不会成为僵尸进程，同时父进程也不被阻塞。

## 8.10 函数exec

用fork创建新的子进程后，子进程要调用一种exec函数以执行另一个的程序，新程序从main开始执行。exec只是用磁盘上的一个新程序替换了当前进程的正文段、数据段、堆段和栈段。

<u>用fork创建新进程，用exec可以初始执行新程序，用exit和wait处理终止和等待终止，这些是我们需要的基本进程控制原语。</u>

![](E:\EBOOK\GITnote\APUE_3rd\APUE_pics\image-20210707223107853.png)

# 进程关系

## 9.4 进程组

每个进程除了有进程ID之外，还属于一个进程组，组长进程的进程组ID等于其进程ID。进程组组长可以创建一个进程组、创建该组中的进程，然后终止。只要进程组中有一个进程存在，则进程组就存在，与组长进程是否终止无关。

```c
#include <unistd.h>
pid_t getpgrp(void);	
int setpgid(pid_t pid, pid_t pgid);
```

`getpgrp`可以返回调用进程的进程组ID。`setpgid`将pid进程的进程组ID设置为`pgid`，如果两个id相等则由`pid`指定的进程变成进程组组长，如果`pid`是0，则使用调用者的进程ID，如果`pgid`是0，则由pid指定的进程ID用作进程组ID。一个进程只能为自己或者它的子进程设置进程组ID，当子进程调用了`exec`后，就不能再更改子进程的进程组ID。

## 9.5 会话

会话（session）是一个或多个进程组的集合。

```c
#include <unistd.h>
pid_t setsid(void);
pid_t getsid(pid_t pid);
```

进程调用`setsid`函数建立一个新会话。如果调用进程不是一个进程组的组长，则此函数创建一个新会话，这时会发生3件事：

- 该进程变成新会话的会话首进程（session leader，会话首进程是创建该会话的进程）。此时，该进程是新会话中的唯一进程。
- 该进程成为一个进程组的组长进程，新进程组ID是该调用进程的进程ID。
- 该进程没有控制终端。如果在调用`setsid`之前该进程有一个控制终端，那么这种联系也被切断。

`getsid`返回会话首进程的进程组ID。

## 9.6 控制终端

会话和进程组还有一些其他特性。

- 一个会话可以有一个控制终端（controlling terminal）。在终端登录情况下是终端设备，在网络登录情况下是伪终端设备。
- 建立与控制终端连接的会话首进程被称为控制进程（controlling process）。
- 一个会话中的几个进程组可被分成一个前台进程组（foreground process group）以及一个或多个后台进程组（background process group）。
- 如果一个会话有一个控制终端，则它有一个前台进程组，其他进程组为后台进程组。
- 无论何时键入终端的退出键（ctrl+\）或中断键（Delete或ctrl+c），都会将退出信号或中断信号发送至前台进程组的所有进程。
- 如果终端接口检测到调制解调器或网络断开连接，则将挂断信号发送至控制进程（会话首进程）。

![](E:\EBOOK\GITnote\APUE_3rd\APUE_pics\image-20210715112349347.png)

通常登录时将自动建立控制终端。

# 信号

## 10.3 函数signal

```c
#include <signal.h>
void (*signal(int signo, void (*func)(int)))(int);
```

signo是信号名，func值是`SIG_IGN`、`SIG_DFL`、接到此信号后要调用的函数的地址。信号发生时，我们称调用函数为捕捉该信号，称调用的函数为信号处理程序（signal handler）或信号捕捉函数（signal-catching function）。

signal函数原型说明此函数要求两个参数，返回一个函数指针，而该指针指向的函数无返回值。The signal function’s first argument, `signo`, is an integer. <u>The second argument is a pointer to a function that takes a single integer argument and returns nothing.</u> The function whose address is returned as the value of signal takes a single integer argument (the final (int)). In plain English, this declaration says that the **signal handler** is passed a single integer argument (the signal number) and that it returns nothing. <u>When we call signal to establish the signal handler, the second argument is a pointer to the function. The return value from signal is the pointer to the previous signal handler.</u>

以上signal函数原型太复杂了，可以使用typedef让其简单一些 `typedef void Sigfunc(int);` 然后 `Sigfunc *signal(int, Sigfunc *);` 

## 10.11 信号集

```c
#include <signal.h>
int sigemptyset(sigset_t *set);
int sigfillset(sigset_t *set);
int sigaddset(sigset_t *set, int signo);
int sigdelset(sigset_t *set, int signo);
```

函数`sigemptyset`初始化由`set`指向的信号集，清除其中的所有信号。函数`sigfillset`初始化由`set`指向的信号集，使其包括所有的信号。所有应用程序在使用信号集前，要对该信号集调用这两个函数的其中之一一次。一旦初始化了一个信号集后，就可以用`sigaddset`和`sigdelset`在信号集中增删特定信号。

## 10.12 函数sigprocmask

该函数可以检测或更改，或同时检测和更改进程的信号屏蔽字。

```c
#include <signal.h>
int sigprocmask(int how, const sigset_t *restrict set, sigset_t *restrict oset);
```

若oset是一个非空指针，则进程的当前信号屏蔽字通过oset返回。若set是一个非空指针，则how指示如何修改当前信号屏蔽字，how可以设置为：SIG_BLOCK（或操作）、SIG_UNBLOCK、SIG_SETMASK（赋值操作）。

## 10.14 函数sigaction

```c
#include <signal.h>
int sigaction(int signo, const struct sigaction *restrict act,
              struct sigaction *restrict oact);
```

该函数的功能是检查或修改与指定信号相关联的处理动作。`signo`是要检测或修改动作的信号编号，若`act`指针非空，则修改其动作，若`oact`指针非空，则系统经由`oact`指针返回该信号的上一个动作。此函数使用下列结构：

```c
struct sigaction {
    void (*sa_handler)(int); 
    sigset_t sa_mask; 			
    int sa_flags; 				
    /* alternate handler */
    void (*sa_sigaction)(int, siginfo_t *, void *);
};
```

当更改信号动作时，如果`sa_handler`字段包含一个信号捕捉函数的地址（不是常量`SIG_IGN`或`SIG_DFL`），则`sa_mask`字段说明了一个信号集，处理该信号时将阻塞信号集中包含的信号。`sa_flags`取0表示默认操作。

```c
/*用sigaction实现的signal函数*/
#include "apue.h"
/* Reliable version of signal(), using POSIX sigaction(). */
Sigfunc *
    signal(int signo, Sigfunc *func)
{
    struct sigaction act, oact;
    act.sa_handler = func;
    sigemptyset(&act.sa_mask);
    act.sa_flags = 0;
    if (signo == SIGALRM) {
        #ifdef SA_INTERRUPT
        act.sa_flags |= SA_INTERRUPT;
        #endif
    } else {
        act.sa_flags |= SA_RESTART;
    }
    if (sigaction(signo, &act, &oact) < 0)
        return(SIG_ERR);
    return(oact.sa_handler);
}
```

# 线程

## 11.3 线程标识

```c
#include <pthread.h>
int pthread_equal(pthread_t tid1, pthread_t tid2);
pthread_t pthread_self(void);
```

进程ID用pid_t表示，是一个非负整数，pthread_t是一个结构，不能当做整数处理，所以必须使用一个函数对两个线程ID进行比较，通过调用pthread_self获得自身的线程ID。

## 11.4 线程创建

```c
#include <pthread.h>
int pthread_create(pthread_t *restrict tidp, 
                   const pthread_attr_t *restrict attr, 
                   void *(*start_rtn)(void *), 
                   void *restrict arg);
```

**当pthread_create成功返回后**，**tidp是新创建的线程ID指向的内存单元**，attr是线程属性，默认为null。新创建的线程从start_rtn函数的地址开始运行，如果需要向start_rtn函数传递的参数有一个以上，那么需要把这些参数放到一个结构中，然后把这个结构的地址作为arg参数传入。

## 11.5 线程终止

```c
#include <pthread.h>
void pthread_exit(void *rval_ptr);
int pthread_join(pthread_t thread, void **rval_ptr);
```

单个线程有3种退出方式，可以再不终止整个进程的情况下停止它的控制流：

1. The thread can simply return from the start routine. The return value is the thread’s exit code.
2. The thread can be canceled by another thread in the same process.
3. The thread can call pthread_exit.

调用pthread_join可以等待指定的线程终止，可以把rval_ptr设置为NULL，不获取线程的终止状态。如果线程从它的启动例程返回，rval_ptr就包含返回码。如果线程被取消，由rval_ptr指定的内存单元就被设置为PTHREAD_CANCELED。

<u>当一个线程通过调用pthread_exit退出或简单地从启动例程返回，进程中的其他线程可以通过调用pthread_join函数获得该线程退出状态。</u>

```c
#include <pthread.h>
int pthread_cancel(pthread_t tid);
void pthread_cleanup_push(void (*rtn)(void *), void *arg);
void pthread_cleanup_pop(int execute);
```

pthread_cancel并不等待线程终止，仅仅提出请求。

线程可以安排它退出时需要调用的函数，这与进程退出时的atexit函数是类似的，所调用的函数称为线程清理处理程序（thread cleanup handler），一个线程可以建立多个清理处理程序，处理程序记录在栈中，也就是说，它们的执行顺序和注册顺序相反。

如果线程是通过从它的启动例程中返回而终止的话，它的清理处理程序就不会被调用，调用pthread_exit就会调用。

![](E:\EBOOK\GITnote\APUE_3rd\APUE_pics\image-20210708230556119.png)

默认情况下，线程的终止状态会保存直到对该线程调用pthread_join。如果线程已经被分离，线程的底层存储资源会在线程终止时立即收回。在线程被分离后，对分离状态的线程调用pthread_join是未定义行为。

## 11.6 线程同步

### 11.6.1 互斥量

互斥量（mutex）从本质上说是一把锁，在访问共享资源前对互斥量进行设置，在访问完成后释放互斥量。对互斥量加锁后，任何其他试图再次对互斥量加锁的额线程都会被阻塞，知道当前线程释放该互斥锁。如果释放互斥量时有一个以上的线程阻塞，那么所有该锁上的阻塞线程都会变成可运行状态，第一个变为运行的线程就可以对互斥量加锁，其他线程就会看到互斥量依然是锁着的，只能回去等待它重新变为可用。这种方式下，每次只有一个线程可以向前执行。

```c
#include <pthread.h>
int pthread_mutex_init(pthread_mutex_t *restrict mutex,
                       const pthread_mutexattr_t *restrict attr);
int pthread_mutex_destroy(pthread_mutex_t *mutex);
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```

在使用互斥量以前，必须对它进行初始化，可以把它设置为常量`PTHREAD_MUTEX_INITIALIZER`，也可以通过调用`pthread_mutex_init`函数进行初始化。如果是动态分配的互斥量（例如使用`malloc`函数），在释放内存前需要调用`pthread_mutex_destroy`。要用默认的属性初始化互斥量，只需要把`attr`设为`NULL`。

对互斥量进行加锁，需要调用`pthread_mutex_lock`。对互斥量解锁，需要调用`pthread_mutex_unlock`。如果线程不希望被阻塞，它可以使用`pthread_mutex_trylock`尝试对互斥量进行加锁，如果未锁则会锁住互斥量并返回0，如果已锁则会返回`EBUSY`。

```c
#include <stdlib.h>
#include <pthread.h>
#define NHASH 29
#define HASH(id) (((unsigned long)id)%NHASH)
struct foo *fh[NHASH];
pthread_mutex_t hashlock = PTHREAD_MUTEX_INITIALIZER; 	// 静态分配的互斥量这样初始化
														// 动态分配的用_init()函数初始化
struct foo {
    int f_count;
    pthread_mutex_t f_lock;
    int f_id;
    struct foo *f_next; /* protected by hashlock */
    /* ... more stuff here ... */
};
struct foo *foo_alloc(int id) /* allocate the object */
{
    struct foo *fp;
    int idx;
    if ((fp = malloc(sizeof(struct foo))) != NULL) {
        fp->f_count = 1;
        fp->f_id = id;
        if (pthread_mutex_init(&fp->f_lock, NULL) != 0) {
            free(fp);
            return(NULL);
        }
        idx = HASH(id);
        pthread_mutex_lock(&hashlock);
        /*注意这个hash表是怎么链接的*/
        fp->f_next = fh[idx];
        fh[idx] = fp;
        /* 注意这里的先后顺序，在对散列列表的锁解锁之前，先锁定了新对象的互斥量。
        因为新对象是放在全局列表中的，其他线程可以找到它，所以在初始化完成之前要
        阻塞其他线程访问该对象。*/
        pthread_mutex_lock(&fp->f_lock);
        pthread_mutex_unlock(&hashlock);

        /* ... continue initialization ... */
        pthread_mutex_unlock(&fp->f_lock);
    }
    return(fp);
}
void foo_hold(struct foo *fp) /* add a reference to the object */
{
    pthread_mutex_lock(&fp->f_lock);
    fp->f_count++;
    pthread_mutex_unlock(&fp->f_lock);
}
struct foo *foo_find(int id) /* find an existing object */
{
    struct foo *fp;
    pthread_mutex_lock(&hashlock);
    for (fp = fh[HASH(id)]; fp != NULL; fp = fp->f_next) {
        if (fp->f_id == id) {
            /*需要做什么就锁什么，需要搜索的时候就锁住hash，需要访问成员再锁对象*/
            foo_hold(fp);
            break;
        }
    }
    pthread_mutex_unlock(&hashlock);
    return(fp);
}
void foo_rele(struct foo *fp) /* release a reference to the object */
{
    struct foo *tfp;
    int idx;
    /*就是需要锁两次，外边的锁是为直接--引用计数准备的，
    如果是最后引用就还要修改hash，所以要取hash锁。*/
    pthread_mutex_lock(&fp->f_lock); 
    if (fp->f_count == 1) { /* last reference */
        pthread_mutex_unlock(&fp->f_lock);
        /*必须解锁对象才能获得hash锁，在获取hash锁时可能被其他线程阻塞，
        所以需要重新检查判断条件。
        比如其他线程在foo_find就会阻塞本线程，而foo_find后会++引用计数，
        则不再是最后引用，不能修改hash，需要--引用计数。*/
        pthread_mutex_lock(&hashlock);
        pthread_mutex_lock(&fp->f_lock);
        /* need to recheck the condition */
        if (fp->f_count != 1) {
            fp->f_count--;
            pthread_mutex_unlock(&fp->f_lock);
            pthread_mutex_unlock(&hashlock);
            return;
        }
        /* remove from list */
        idx = HASH(fp->f_id);
        tfp = fh[idx];
        if (tfp == fp) {
            // 删链表尾的
            fh[idx] = fp->f_next;
        } else {
            // 删链表中间的
            while (tfp->f_next != fp) tfp = tfp->f_next;
            tfp->f_next = fp->f_next;
        }
        pthread_mutex_unlock(&hashlock);
        pthread_mutex_unlock(&fp->f_lock);
        pthread_mutex_destroy(&fp->f_lock);
        free(fp);
    } else {
        fp->f_count--;
        pthread_mutex_unlock(&fp->f_lock);
    }
}
```

### 11.6.3 函数pthread_mutex_timedlock

该函数和`pthread_mutex_lock`是基本等价的，但是在到达超时时间值时，该函数不会对互斥量尝试加锁，而是返回错误码`ETIMEDOUT`。这是一种避免永久阻塞的方法。

### 11.6.4 读写锁

读写锁有三种状态：读加锁，写加锁，不加锁。一次只有一个线程可以占有写模式的读写锁，但是多个线程可以同时占有读模式的读写锁。写加锁时，所有试图对这个锁加锁的线程都会被阻塞。读加锁时，所有以读模式对它加锁的线程都会得到访问权，但是以写模式对它加锁会阻塞。通常的实现中，当读写锁处于读加锁，这时有一个线程试图写加锁，读写锁通常会阻塞随后的读加锁请求，这样可以避免长期的读加锁，写加锁得不到满足。

读写锁也叫共享互斥锁（shared-exclusive lock）。读加锁被看作以共享模式锁住，写加锁被看作以互斥模式锁住。

```c
#include <pthread.h>
int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock,
                        const pthread_rwlockattr_t *restrict attr);
int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);
```

### 11.6.6 条件变量

```c
#include <pthread.h>
int pthread_cond_init(pthread_cond_t *restrict cond, const pthread_condattr_t *restrict attr);
int pthread_cond_destroy(pthread_cond_t *cond);
int pthread_cond_wait(pthread_cond_t *restrict cond, pthread_mutex_t *restrict mutex);
int pthread_cond_timedwait(pthread_cond_t *restrict cond, pthread_mutex_t *restrict mutex,
                           const struct timespec *restrict tsptr);
```

使用跟互斥量一样。有两个函数可以通知线程条件已经满足。`pthread_cond_signal`能缓行一个等待该条件的线程，`pthread_cond_broadcast`能唤醒等待该条件的所有线程。在调用这两个函数时，我们说这是在给线程或者条件**发信号**。一定要在改变状态条件以后再给线程发信号。

```c
#include <pthread.h>
int pthread_cond_signal(pthread_cond_t *cond);
int pthread_cond_broadcast(pthread_cond_t *cond);
```

### 11.6.7 自旋锁

自旋锁可用于：锁被持有的时间短，而且线程不希望在重新调度上花费太多成本。自旋锁与互斥量类似，但它不通过休眠使进程阻塞，而是在获取锁之前一直处于忙等（busy-waiting）。

### 11.6.7 屏障

```c
#include <pthread.h>
int pthread_barrier_init(pthread_barrier_t *restrict barrier, 
                         const pthread_barrierattr_t *restrict attr,
                         unsigned int count);
int pthread_barrier_destroy(pthread_barrier_t *barrier);
int pthread_barrier_wait(pthread_barrier_t *barrier);
```

调用`pthread_barrier_wait`的线程在屏障（调用`pthread_barrier_init`时设定的`count`数）计数未满足时，会进入休眠状态。如果该线程是最后一个调用`pthread_barrier_wait`的线程，就满足了屏障计数，所有的线程都被唤醒。

# 线程控制

## 12.3 线程属性

```c
#include <pthread.h>
int pthread_attr_init(pthread_attr_t *attr);
int pthread_attr_destroy(pthread_attr_t *attr);
int pthread_attr_getdetachstate(const pthread_attr_t *restrict attr, int *detachstate);
int pthread_attr_setdetachstate(pthread_attr_t *attr, int detachstate);
```

如果对现有的某个线程的终止状态不感兴趣的话，可以用`pthread_detach`函数让操作系统在线程退出时收回它所占用的资源。如果在创建线程时就知道不需要了解线程的终止状态，就可以修改`pthread_attr_t`结构中的`detachstate`属性，让线程从一开始就处于分离状态。可以使用`pthread_attr_setdetachstate`函数把`detachstate`设为两个值之一：`PTHREAD_CREATE_DETACHED`，以分离状态启动线程；`PTHREAD_CREATE_JOINABLE`，正常启动线程。

## 12.4 同步属性

### 12.4.1 互斥量属性

```c
#include <pthread.h>
int pthread_mutexattr_init(pthread_mutexattr_t *attr);
int pthread_mutexattr_destroy(pthread_mutexattr_t *attr);
```

互斥量的默认属性是通过使用`PTHREAD_MUTEX_INITIALIZER`常量或调用`pthread_mutex_init`得到，非默认属性则要通过`pthread_mutexattr_init`初始化`pthread_mutexattr_t`结构。值得注意的3个属性是：进程共享属性（process-shared attribute）、健壮属性（robust attribute）、类型属性（type attribute）。

在进程中，多个进程可以访问同一个同步对象，这时是`PTHREAD_PROCESS_PRIVATE`。存在这样的机制：允许相互独立的多个进程把同一个内存数据块映射到它们各自独立的地址空间中。这时多个进程访问共享数据也需要同步，设置`PTHREAD_PROCESS_SHARED`，从共享内存数据块中分配的互斥量就可以用于这些进程的同步。

使用健壮的互斥量改变了我们使用`pthread_mutex_lock`的方式，默认情况下锁后行为是`PTHREAD_MUTEX_STALLED`，这是未定义的，而`PTHREAD_MUTEX_ROBUST`可以判断出，当锁被进程获取后没有解锁便终止的情况。

类型互斥量控制着互斥量的锁定属性：

- `PTHREAD_MUTEX_NORMAL`：标准互斥量类型，不做错误检查或死锁检查。
- `PTHREAD_MUTEX_ERRORCHECK`：提供给错误检查。
- `PTHREAD_MUTEX_RECURSIVE`：递归互斥量，维护加锁计数。
- `PTHREAD_MUTEX_DEFAULT`：默认，映射为以上三种之一。

## 12.5 Reentrancy

如果一个函数在相同的时间点可以被多个线程安全地调用（一个函数对多个线程来说是可重入的（reentrant）），就称该函数是线程安全的。

## 12.6 线程特定数据

thread-specific data，也称为线程私有数据（thread-private data），是存储和查询某个特定线程相关数据的一种机制。我们希望每个线程都可以访问它自己单独的数据副本，而不需要担心与其他线程的同步访问问题。

```c
#include <pthread.h>
int pthread_key_create(pthread_key_t *keyp, void (*destructor)(void *));
```

线程通常使用`malloc`为线程特定数据分配内存，在分配之前，需要创建与该数据关联的键。创建的键存储在`keyp`指向的内存单元中，这个键可以被进程中的所有线程使用，但每个线程把这个键与不同的线程特定数据地址进行关联。第二个参数可以关联一个可选的析构函数，当线程调用`pthread_exit`或线程执行返回，析构函数就被会调用，防止线程在没有释放内存之前就退出导致内存泄露。

## 12.8 线程和信号

```c
#include <signal.h>
int pthread_sigmask(int how, const sigset_t *restrict set, sigset_t *restrict oset);
```

进程中的信号是递送到单个线程的。10.12节的`sigprocmask`函数行为在多线程进程中没有定义，所以必须使用`pthread_sigmask`。

# 守护进程

## 13.3 编程规则

编写守护进程程序的规则：

1. 调用umask将文件模式创建屏蔽字设置为一个已知值（0）。这样做使读写权限无屏蔽，在fork过程中子进程继承父进程的文件权限屏蔽字，防止继承得到的文件模式读写失效。
2. 调用fork，然后使父进程exit。这样做实现了：第一，如果该守护进程是作为shell命令启动的，父进程终止会让shell认为这条指令已经执行完毕。第二，子进程继承了父进程的进程组ID，但获得了一个新的进程ID，这是使用setsid的先决条件。
3. 调用`setsid`创建一个新会话。这样使守护进程没有控制终端。
4. 将当前工作目录更改为根目录。守护进程在系统重启前是一直存在的，要保证守护进程的工作目录一直存在。
5. 关闭不再需要的文件描述符。关闭从父进程继承来的任何fd，可以调用`open_max`函数（2.17节）或`getrlimit`（7.11节）判定最大fd值，并关闭直到该值的所有fd。如果不关闭的话会浪费资源。
6. 打开`/dev/null`使其具有文件描述符0，1，2。文件描述符 0，1，2在改变之前指的是读标准输入、写标准输出和标准错误，这样切断了守护进程与终端的联系。

```c
#include "apue.h"
#include <syslog.h>
#include <fcntl.h>
#include <sys/resource.h>
void daemonize(const char *cmd)
{
    int i, fd0, fd1, fd2;
    pid_t pid;
    struct rlimit rl;
    struct sigaction sa;
    /*
    * Clear file creation mask.
    * 修改文件模式创建屏蔽字，防止用户权限受限。
    */
    umask(0);
    /*
    * Get maximum number of file descriptors.
    * 获得最大fd。
    */
    if (getrlimit(RLIMIT_NOFILE, &rl) < 0)
        err_quit("%s: can’t get file limit", cmd);
    /*
    * Become a session leader to lose controlling TTY.
    * 第一次fork，1·让shell认为命令执行完毕，不用挂在终端输入上。
    * 2·达成调用setsid的前提，子进程成为新会话组长。
    */
    if ((pid = fork()) < 0)
        err_quit("%s: can’t fork", cmd);
    else if (pid != 0) /* parent */
        exit(0);
    setsid();
    /*
    * Ensure future opens won’t allocate controlling TTYs.
    * sigaction参见10.14节。此处为了忽略与控制终端联系的信号，
    * 第二次fork不是必须的，这是为了防止再次打开控制终端。
    * 因为打开终端的前提是会话组长，父进程是会话组长，所以fork后更保险。
    */
    sa.sa_handler = SIG_IGN;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    if (sigaction(SIGHUP, &sa, NULL) < 0)
        err_quit("%s: can’t ignore SIGHUP", cmd);
    if ((pid = fork()) < 0)
        err_quit("%s: can’t fork", cmd);
    else if (pid != 0) /* parent */
        exit(0);
    /*
    * Change the current working directory to the root so
    * we won’t prevent file systems from being unmounted.
    */
    if (chdir("/") < 0)
        err_quit("%s: can’t change directory to /", cmd);
    /*
    * Close all open file descriptors.
    */
    if (rl.rlim_max == RLIM_INFINITY)
        rl.rlim_max = 1024;
    for (i = 0; i < rl.rlim_max; i++)
        close(i);
    /*
    * Attach file descriptors 0, 1, and 2 to /dev/null.
    */
    fd0 = open("/dev/null", O_RDWR);
    fd1 = dup(0);
    fd2 = dup(0);
    /*
    * Initialize the log file.
    */
    openlog(cmd, LOG_CONS, LOG_DAEMON);
    if (fd0 != 0 || fd1 != 1 || fd2 != 2) {
        syslog(LOG_ERR, "unexpected file descriptors %d %d %d",
               fd0, fd1, fd2);
        exit(1);
    }
}
```

# 高级IO

## 14.2 非阻塞IO

对于一个给定的描述符，有两种为其制定非阻塞IO的方法：

- 如果调用open获得描述符，则可指定`O_NONBLOCK`标志。
- 对于一个已经打开的描述符，则可调用fcntl，由该函数打开`O_NONBLOCK`文件状态标志。

非阻塞IO会多次调用IO函数，然而其中大多数都会因本应阻塞而产生错误返回，这种轮询会浪费CPU时间。

## 14.3 记录锁

记录锁的功能是：当第一个进程正在读写文件的某个部分时，使用记录锁可以阻止其他进程修改同一文件区。对于UNIX而言，“record”是一种误用，因为UNIX内核没有使用文件记录的概念，更合适的术语可能是**字节范围锁**（byte-range locking），因为它锁定的只是文件的一个区域。

```c
#include <fcntl.h>
int fcntl(int fd, int cmd, ... /* struct flock *flockptr */ );
```

使用fcntl函数设置记录锁。对于记录锁，cmd是`F_GETLK`、`F_SETLK`、`F_SETLKW`。第三个参数是一个指向flock结构的指针。

```c
struct flock {
    short l_type; /* F_RDLCK, F_WRLCK, or F_UNLCK */
    short l_whence; /* SEEK_SET, SEEK_CUR, or SEEK_END */
    off_t l_start; /* offset in bytes, relative to l_whence */
    off_t l_len; /* length, in bytes; 0 means lock to EOF */
    pid_t l_pid; /* returned with F_GETLK */
};
```

- `l_type`，所希望的锁类型：`F_RDLCK`共享读锁，`F_WRLCK`独占性写锁，`F_UNLCK`解锁一个区域
- `l_start`，`l_whence`，加锁或解锁区域的起始字节偏移量
- `l_len`，The size of the region in bytes 
- `l_pid`，（仅由F_GETLK返回）进程ID持有的锁能阻塞当前进程

关于记录锁的自动继承和释放：

1. 锁与进程和文件两者相关联。进程终止，则该进程建立的锁被全部释放；fd关闭，则通过fd设置的锁全部释放。
2. 由fork产生的子进程不继承父进程所设置的锁，子进程需要调用fcntl才能获得从父进程继承来fd的锁。
3. 在执行exec后，新程序可以继承原执行程序的锁。

## 14.4 IO多路转接

### 14.4.1 函数select和pselect

```c
#include <sys/select.h>
int select(int maxfdp1, fd_set *restrict readfds,
           fd_set *restrict writefds, fd_set *restrict exceptfds,
           struct timeval *restrict tvptr);
int FD_ISSET(int fd, fd_set *fdset);
											Returns: nonzero if fd is in set, 0 otherwise
void FD_CLR(int fd, fd_set *fdset);
void FD_SET(int fd, fd_set *fdset);
void FD_ZERO(fd_set *fdset);
```

`tvptr`指定愿意等待的时间长度，`readfds`、`writefds`、`exceptfds`是指向fd集的指针，这是我们关心的可读、可写、处于异常条件的fd集合。每个fd集存储在一个`fd_set`数据类型中。`FD_ZERO`将一个fd_set变量的所有位设置为0。`FD_SET`可以开启一位，`FD_CLR`可以清除一位。最后，可以调用`FD_ISSET`测试fd集中一个指定位是否打开。select的中间三个参数可以全部是空指针，表示对相应条件都不关心，这时select就成为一个比sleep更精准的定时器。`maxfdp1`表示“最大fd值+1”，对一般程序来说不需要达到最大fd，只需要设定需要关注的最大fd范围，在小于`maxfdp1`内寻找打开的位。

### 14.4.2 函数poll

```c
#include <poll.h>
int poll(struct pollfd fdarray[], nfds_t nfds, int timeout);
						Returns: count of ready descriptors, 0 on timeout, −1 on error;
struct pollfd {
    int fd; /* file descriptor to check, or <0 to ignore */
    short events; /* events of interest on fd */
    short revents; /* events that occurred on fd */
};                                            
```

与select不同，poll不是为每个条件构造一个fd集，而是构造一个pollfd结构的数组，每个数组元素指定一个fd以及我们对该fd感兴趣的条件。

## 14.5 异步IO

```c
struct aiocb {
    int 			aio_fildes; /* file descriptor */
    off_t 			aio_offset; /* file offset for I/O */
    volatile void 	*aio_buf; /* buffer for I/O */
    size_t 			aio_nbytes; /* number of bytes to transfer */
    int 			aio_reqprio; /* priority */
    struct sigevent aio_sigevent; /* signal information */
    int 			aio_lio_opcode; /* operation for list I/O */
};
struct sigevent {
    int 			sigev_notify; /* notify type */
    int 			sigev_signo; /* signal number */
    union sigval 	sigev_value; /* notify argument */
    void (*sigev_notify_function)(union sigval); /* notify function */
    pthread_attr_t 	*sigev_notify_attributes; /* notify attrs */
};
```

aio_sigevent字段控制，在IO完成后，如何通知应用程序。

# 14.6 函数readv和writev

```c
#include <sys/uio.h>
ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);
					Both return: number of bytes read or written, −1 on error
struct iovec {
    void *iov_base; /* starting address of buffer */
    size_t iov_len; /* size of buffer */
};
```

这两个函数用于在一次函数调用中读写多个非连续缓冲区，也称为散布读（scatter read）和聚集写（gather write）。`iov`数组中的元素数由`iovcnt`指定，最大值受限于`IOV_MAX`（2.5节）。writev从缓冲区中聚集输出数据的顺序是：`iov[0]...iov[iovcnt-1]`，返回输出的字节总数等于所有缓冲区长度之和。readv则将读入的数据按同样顺序散步到缓冲区中，先填满一个缓冲区再填写下一个，如果遇到文件尾，则返回0。

## 14.7 函数readn和writen

在管道、FIFO和终端、网络中，一次read/write返回的数据可能少于指定的字节数，这两个函数就是为了处理返回值小于要求值的情况。

```c
#include "apue.h"
ssize_t readn(int fd, void *buf, size_t nbytes);
ssize_t writen(int fd, void *buf, size_t nbytes);
							Both return: number of bytes read or written, −1 on error
```

## 14.8 存储映射IO

存储映射IO（memory-mapped IO）能将一个磁盘文件映射到存储空间中的一个缓冲区上。从缓冲区取数据就相当于读文件，写缓冲区相应字节就自动写入文件。

```c
#include <sys/mman.h>
void *mmap(void *addr, size_t len, int prot, int flag, int fd, off_t off );
					Returns: starting address of mapped region if OK, MAP_FAILED on error
```

addr指定映射存储区的起始地址，通常为0，表示由系统选择映射区起始地址，函数返回该映射区的起始地址。fd指定被映射文件，映射之前必须先打开该文件。len指定映射字节数，off指定映射字节在文件中的偏移量。prot指定了映射存储区的保护要求。

![](E:\EBOOK\GITnote\APUE_3rd\APUE_pics\image-20210718153419677.png)

# 进程间通信

（interprocess communication，IPC）

## 15.2 管道

![](C:\Users\hasee\AppData\Roaming\Typora\typora-user-images\image-20210718211129037.png)

```c
#include <unistd.h>
int pipe(int fd[2]);
															Returns: 0 if OK, −1 on error
```

fd[0]作为读，fd[1]作为写。

## 15.3 函数popen和pclose

```c
#include <stdio.h>
FILE *popen(const char *cmdstring, const char *type);
										Returns: file pointer if OK, NULL on error
int pclose(FILE *fp);
									Returns: termination status of cmdstring, or −1 on error
```

popen先执行fork，然后调用exec执行cmdstring，并且返回一个标准IO文件指针。如果type是“r”，则返回文件的可读指针，如果是“w”，则是可写。cmdstring由shell这样执行：`sh -c cmdstring`，例如：`fp = popen("ls *.c","r");`

## 15.5 FIFO

FIFO被称为named pipes，unnamed pipes只能在两个相关的进程之间使用（这两个相关进程有一个共同的祖先进程），通过FIFO，不相关的进程也可以交换数据。

```c
#include <sys/stat.h>
int mkfifo(const char *path, mode_t mode);
int mkfifoat(int fd, const char *path, mode_t mode);
													Both return: 0 if OK, −1 on error
```

FIFO有两种用途：

1. shell命令使用FIFO将数据从一条管道传送到另一条时，无需创建中间文件。
2. 客户进程-服务器进程应用程序中，FIFO用作汇聚点（rendezvous point），在客户进程和服务器进程之间传递数据。

# 网络IPC：套接字

## 16.2 套接字描述符

```c
#include <sys/socket.h>
int socket(int domain, int type, int protocol);
										Returns: file (socket) descriptor if OK, −1 on error
```

domain：

- AF_INET：IPv4 Internet domain
- AF_INET6：IPv6 Internet domain (optional in POSIX.1)
- AF_UNIX：UNIX domain
- AF_UNSPEC：unspecified

type：

- SOCK_DGRAM：固定长度的、无连接的、不可靠的报文传递
- SOCK_RAW：IP协议的数据报接口 (optional in POSIX.1)
- SOCK_SEQPACKET：固定长度的、有序的、可靠的、面向连接的报文传递
- SOCK_STREAM：有序的、可靠的、双向的、面向连接的字节流

protocol：

- IPPROTO_IP：IPv4 Internet Protocol
- IPPROTO_IPV6：IPv6 Internet Protocol (optional in POSIX.1)
- IPPROTO_ICMP：Internet Control Message Protocoly  因特网控制报文协议
- IPPROTO_RAW：Raw IP packets protocol (optional in POSIX.1)  原始IP数据包协议
- IPPROTO_TCP：Transmission Control Protocol  传输控制协议
- IPPROTO_UDP：User Datagram Protocol  用户数据报协议

调用`socket`与调用`open`类似，均可获得用于IO的文件描述符，当不再需要该文件描述符，调用`close`关闭该访问。

```c
#include <sys/socket.h>
int shutdown(int sockfd, int how);
										Returns: 0 if OK, −1 on error
```

调用`shutdown`来禁止套接字的IO，可以选择`SHUT_RD`、`SHUT_WR`、`SHUT_RDWR`。

## 16.3 寻址

### 16.3.1 字节序

对于TCP/IP程序，有4个函数用于处理处理器字节序与网络字节序的转换。

```c
#include <arpa/inet.h>
uint32_t htonl(uint32_t hostint32);
					Returns: 32-bit integer in network byte order
uint16_t htons(uint16_t hostint16);
					Returns: 16-bit integer in network byte order
uint32_t ntohl(uint32_t netint32);
					Returns: 32-bit integer in host byte order
uint16_t ntohs(uint16_t netint16);
					Returns: 16-bit integer in host byte order
```

The `h` is for ‘‘host’’ byte order, and the `n` is for ‘‘network’’ byte order. The `l` is for ‘‘long’’ (i.e., 4-byte) integer, and the `s` is for ‘‘short’’ (i.e., 2-byte) integer.

### 16.3.2 地址格式

```c
#include <arpa/inet.h>
const char *inet_ntop(int domain, const void *restrict addr, 
                     char *restrict str, socklen_t size);
									Returns: pointer to address string on success, NULL on error
int inet_pton(int domain, const char *restrict str,
             void *restrict addr);
									Returns: 1 on success, 0 if the format is invalid, or −1 on error
```

`p` is for  presentation，`n` is for numeric。`inet_ntop`将网络字节序的二进制地址转换成文本字符串格式。`inet_pton`将文本字符串格式转换成网络字节序的二进制地址。参数`domain`仅支持`AF_INET`和`AF_INET6`。

linux中，`sockaddr_in`定义如下

```c
struct sockaddr_in{
    sa_family_t		sin_family;		// address family
    in_port_t 		sin_port;		// port number
    struct in_addr	sin_addr;		// ipv4 address
    unsigned char	sin_zero[8];	// filler POSIX标准中没有此段
}
struct in_addr{
    in_addr_t		s_addr;			// ipv4 address
}
```

### 16.3.3 地址查询

```c
#include <sys/socket.h>
#include <netdb.h>
int getaddrinfo(const char *restrict host, const char *restrict service,
                const struct addrinfo *restrict hint, 
                struct addrinfo **restrict res);
													Returns: 0 if OK, nonzero error code on error
void freeaddrinfo(struct addrinfo *ai);

int getnameinfo(const struct sockaddr *restrict addr, socklen_t alen,
                char *restrict host, socklen_t hostlen,
                char *restrict service, socklen_t servlen, int flags);
													Returns: 0 if OK, nonzero on error
```

`getaddrinfo`函数将一个主机名和一个服务名映射到一个地址。`getnameinfo`将一个地址转换成一个主机名和一个服务名。

### 16.3.4 将套接字与地址关联

```c
#include <sys/socket.h>
int bind(int sockfd, const struct sockaddr *addr, socklen_t len);
												Returns: 0 if OK, −1 on error
```

## 16.4 建立连接

```c
#include <sys/socket.h>
int connect(int sockfd, const struct sockaddr *addr, socklen_t len);
											Returns: 0 if OK, −1 on error
```

如果要处理一个面向连接的网络服务（`SOCK_STREAM`或`SOCK_SEQPACKET`），那么在交换数据以前，需要在请求服务的进程套接字（客户端）和提供服务的进程套接字（服务器）之间利用`connect`建立一个连接。

