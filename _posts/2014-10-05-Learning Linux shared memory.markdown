---
layout: post
title: 学习使用Linux共享内存
categories:
- Linux
tags:
- Linux,shared memory
---

##共享内存使用步骤
1. shmget() 根据key创建或者获取共享内存的shmid     
2. shmat()   连接attach，把共享内存映射到本进程的地址空间，允许本进程使用共享内存
3. 读写共享内存...
4. shmdt() 脱离detach，禁止本进程使用共享内存
5. shmctl(sid,IPC_RMID,0) 标记删除共享内存。也可以使用ipcrm命令删除

*共享内存大小限制# cat /proc/sys/kernel/shmmax，单位：字节。*

![共享内存地址空间](/pic/shared_memory.jpg)


##`int shmget(key_t key, size_t size, int shmflg);`
**参数：**  
key：进程间事先约定的key，或者调用`key_t ftok( char * fname, int id )`获取  
size：内存大小，字节。（当创建一个新的共享内存区时，size 的值必须大于 0 ；如果是访问一个已经存在的内存共享区，则 size 可以是 0 。）  
shmflg：标志位IPC_CREATE、IPC_EXCL

* IPC_CREATE : 调用 shmget 时，系统将此值与其他共享内存区的 key 进行比较。如果存在相同的 key ，说明共享内存区已存在，此时返回该共享内存区的shmid；如果不存在，则新建一个共享内存区并返回其shmid。
* IPC_EXCL : 该宏必须和 IPC_CREATE 一起使用，否则没意义。当 shmflg 取 IPC_CREATE | IPC_EXCL 时，表示如果发现内存区已经存在则返回 -1，错误代码为 EEXIST 。

返回值：成功返回shmid；失败返回-1

一般我们创建共享内存的时候会在第一个进程中使用shmget来创建共享内存。调用`int shmid = shmget(key, size, IPC_CREATE|0666);`  
而在其它进程中使用shmget和同样的key来获取到这个已经创建了的共享内存。也还是调用`int shmid = shmget(key, size, IPC_CREATE|0666);`


##`void *shmat(int shmid, const void *shmaddr, int shmflg);`
**参数：**  
shmid：shmget的返回值  
shmaddr：共享内存连接到本进程后在本进程地址空间的内存地址。如果为NULL，则由内核选择一块空闲的地址。如果非空，则由该参数指定地址。一般传入NULL。  
shmflg：一般为 0 ，则不设置任何限制权限。  

* SHM_RDONLY 只读
* SHM_RND 如果指定了shmaddr，那么设置该标志位后，连接的内存地址会对SHMLBA向下取整对齐。
* SHM_REMAP   /\* take-over region on attach \*/ 不清楚

返回值：成功后返回一个指向共享内存区的指针；否则返回 `(void *) -1`

通常使用 `void ptr = shmat(shmid, NULL,0);`

##`int shmdt(const void *shmaddr);`
**参数：**  
shmaddr：shmat 函数的返回值  
返回值：成功返回0；失败返回-1  

调用shmdt后，共享内存的shmid_ds结构会变化。其中shm_dtime指向当前时间，shm_lpid设置为当前pid，shm_nattch减一。**如果shm_nattch变为0，并且该共享内存已被标记为删除，那么该共享内存会被删除。（也就是说，光调用shmdt，但是没调用删除，就不会删除。并且，只调用删除，没调用shmdt使得nattch=0，也不会删除）**

fork()后，子进程继承父进程连接的共享内存。  
execve()后，子进程detach所有连接的共享内存。  
**_exit()，detach所有共享内存。**

最好不要依赖进程退出来detach，应该由应用程序调用shmdt。因为如果进程异常退出，系统不会detach。


使用ipcs命令可以一直看到挂接数为0的共享内存，这时候用相同的ID去挂接还是能挂接上的，说明这个共享内存一直是存在的。需要调用`shmctl(sid,IPC_RMID,0)`或者执行ipcrm删除。否则会导致相关的内存泄漏。


##其他
`int shmctl(int shmid, int cmd, struct shmid_ds *buf);`  
**参数：**  
cmd  

* IPC_STAT   把shmid_ds从内核中复制到用户缓存
* IPC_SET      
* IPC_RMID  标记删除。只有当nattch=0时才会真正删除。被标记删除的共享内存shm_perm.mode是**SHM_DEST**。 常用`shmctl(sid,IPC_RMID,0)`
* IPC_INFO  Linux特有。返回系统范围内的共享内存设置
* SHM_INFO, SHM_STAT, SHM_LOCK, SHM_UNLOCK为Linux特有。
    
##关键结构体
<sys/shm.h>
<pre>
           struct shmid_ds {
               struct ipc_perm shm_perm;    /* Ownership and permissions */
               size_t          shm_segsz;   /* Size of segment (bytes) */
               time_t          shm_atime;   /* Last attach time */
               time_t          shm_dtime;   /* Last detach time */
               time_t          shm_ctime;   /* Last change time */
               pid_t           shm_cpid;    /* PID of creator */
               pid_t           shm_lpid;    /* PID of last shmat(2)/shmdt(2) */
               shmatt_t        shm_nattch;  /* No. of current attaches */
               ...
           };
</pre>

<sys/ipc.h>
<pre>
           struct ipc_perm {
               key_t          __key;    /* Key supplied to shmget(2) */
               uid_t          uid;      /* Effective UID of owner */
               gid_t          gid;      /* Effective GID of owner */
               uid_t          cuid;     /* Effective UID of creator */
               gid_t          cgid;     /* Effective GID of creator */
               unsigned short mode;     /* Permissions + SHM_DEST and
                                           SHM_LOCKED flags */
               unsigned short __seq;    /* Sequence number */
           };
</pre>



