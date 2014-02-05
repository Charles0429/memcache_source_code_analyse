##daemonize

在学习`memcached`的进程dameonize之前，我们先要了解`dup`和`dup2`两个函数的使用方法：

###函数简介

`dup`和`dup2`是两个非常有用的调用，他们的作用都是用来复制一个文件的描述符。

其中`dup`函数的声明如下：

    int dup( int oldfd );

返回值

>On success, these system calls return the new descriptor.  On error,
       -1 is returned, and errno is set appropriately.

`dup`函数通常复制文件描述符，比如，可以用把在对标准输入输出进行重定向之前，可以先把标准输入输出的文件描述符保存下来，以便于恢复标准输入输出的重定向。

`dup2`函数的声明如下：

    int dup2(int oldfd, int newfd);

`dup2`的作用是把newfd指向的文件描述符的内容复制成oldfd，这样，就完成了把newfd重定向到oldfd的功能

下面用一个例子来说明两个函数的使用

    #include <unistd.h>
    #include <stdio.h>
    #include <stdlib.h>
    #include <sys/types.h>
    #include <sys/stat.h>
    #include <fcntl.h>
    #include <string.h>
    
    int main(int argc, char *argv)
    {
        int fd;
        int copy_stdout;
        char *msg = "a test message for redirect stdout";
    
        //open a test file to write message
        fd = open("test", O_CREAT | O_RDWR, S_IREAD | S_IWRITE);
    
        //copy the original file descriptor 
        copy_stdout = dup(STDOUT_FILENO);
    
        //redirect the stdout to fd
        dup2(fd, STDOUT_FILENO);
    
        //must close the fd to complete redirect
        //close(fd);
    
        //write the message
        printf("%s\n", msg);
    
        //make sure the message is printed immediately
        fflush(stdout);
    
        //redirect back
        dup2(copy_stdout, STDOUT_FILENO);
    
        //print the message to stdout
        printf("%s\n", msg);
        return 0;    
    }


上面代码中有个需要注意的地方就是`fflush(stdout)`这一行，因为stdio会缓存输出，所以在printf之后要先把缓存清空，立即输出，不然，在后面将标准输出重定向回来之后，就可能将本来输出到文件`test`中的内容一直缓存着，到了标准输出重定向回来再输出，这样就造成`test`文件中没有内容输出了。

###memcache的daemonize

    #include <fcntl.h>
    #include <stdio.h>
    #include <stdlib.h>
    #include <unistd.h>
    
    #include "memcached.h"
    
    int daemonize(int nochdir, int noclose)
    {
        int fd;
    
        switch (fork()) {
        case -1:
            return (-1);
        case 0:
            break;
        default:
            _exit(EXIT_SUCCESS);
        }
    
        if (setsid() == -1)
            return (-1);
    
        if (nochdir == 0) {
            if(chdir("/") != 0) {
                perror("chdir");
                return (-1);
            }
        }
    
        if (noclose == 0 && (fd = open("/dev/null", O_RDWR, 0)) != -1) {
            if(dup2(fd, STDIN_FILENO) < 0) {
                perror("dup2 stdin");
                return (-1);
            }
            if(dup2(fd, STDOUT_FILENO) < 0) {
                perror("dup2 stdout");
                return (-1);
            }
            if(dup2(fd, STDERR_FILENO) < 0) {
                perror("dup2 stderr");
                return (-1);
            }
    
            if (fd > STDERR_FILENO) {
                if(close(fd) < 0) {
                    perror("close");
                    return (-1);
                }
            }
        }
        return (0);
    }

`memcached`的process的daemonize的过程如下：

- 创建一个子进程，并且父进程退出，这么做的原因是新创建的子进程不可能是会话组的首进程的ID，这样保证`setsid()`函数能调用成功
2. 在进程中调用`setsid()`，这个函数完成的功能如下：
>如果调用此函数的进程不是一个进程组的组长，则此函数就会创建一个新会话。此时，该进程是新会话中唯一的进程。
>(1)该进程变成新会话首进程。此时，该进程是新会话中唯一的进程
>(2)该进程成为一个新进程组的组长进程。新进程组ID则是该调用进程的进程ID
>(3)该进程没有控制终端。如果在调用setsid之前该进程有一个控制终端，那么这种联系也会被终端
>如果调用进程已经是一个进程组的组长，则此函数返回出错。为了保证不会发生这种情况，通常先调用fork，然后使其父进程终止，而子进程继续。因为子进程继承了父进程的进程组ID，而其他进程ID则是新分配的，两者不可能相等，所以这就保证子进程不会是一个进程组的组长。

- 把子进程的文件目录设置成根目录
- 重定向标准输入输出和错误，到`/dev/null`设备