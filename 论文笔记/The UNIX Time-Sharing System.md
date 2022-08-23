# 文件系统
## 分类
- 普通文件（Ordinary files）。
- 目录（Directories）
- 特殊文件（Special files）： Each supported I/O device  is associated with at least one such file。

## Removable file systems
mount：There is only one exception to the rule of identical treatment of files on different devices: no link  
may exist between one file system hierarchy and another. This restriction is enforced so as to avoid the  
elaborate bookkeeping that would otherwise be required to assure removal of the links whenever the  
removable volume is dismounted.

## Protection
set-user-ID：当文件的该位有设置时，表示当文件被执行时，程序将具有文件所有者的权限而非执行者的权限。eg. passwd程序。

## I/O syscall
- open
- create
- read
- write
- lseek：随机读写

# 文件系统实现
一个目录项 = 一个文件名 + `i-number` ：i-number is used as an index into a system table (the i-list) stored in a known  
part of the device on which the directory resides。根据 `i-number` 和 `i-list` 找到的节点 `i-node` 包含：
- the user and group-ID of its owner
- its protection bits
- the physical disk or tape addresses for the file contents
- its size
- time of creation, last use, and last modification
- the number of links to the file, that is, the number of times it appears in a directory
- a code indicating whether the file is a directory, an ordinary file, or a special file


### 操作系统层

一个文件被 `open`，文件描述符保存了：`device`、`i-number`、`read/write pointer` 。

一个文件被 `create` ，会分配一个 `i-node` 并分配一个新的 `directory entry` 保存名字和 索引`i-node` 的 `i-number` 到父目录。

创建一个 `link` 文件，需要创建一个新的 `directory entry` 保存名字和 索引`i-node` 的 `i-number` 到父目录，这里只保存了一个 `inode`，需要把 `inode` 中的 `link-count` 增加。

删除文件的话将 `i-number` 对应的 `inode` 的`link-count` 减一，如果 `link-count` 为0，则删除 `i-node` 并释放磁盘资源。

### 磁盘索引层
每个Inode都会预留13个用于索引磁盘块的地址，前10个地址用于一级索引，第11个二级索引地址索引到的磁盘块存放了128个一级索引，后面还有三级索引和四级索引。不同大小的文件，读取时防磁盘次数也会不同（与多级索引有关）。

### 数据块缓存层
事实上，为了防止频繁地访问磁盘，根据空间局部性原理，我们将在内存中暂存一部分常用的磁盘块的数据，当用户访问文件数据时，我们会根据 `i-node` 指示的磁盘块地址查找缓存，如果在缓存中找到目标块，则只需要操作缓存块即可，若未能找到，则需要读取磁盘，并将磁盘块数据加载到缓存中。

### mount实现
mount维护了一个系统表，表里的key存放了 `i-number` 和普通文件名，value保存了与普通文件名对应的设备名。用户使用 `open` 或 `create` 传入文件路径名称，转化为 `i-number/file name` ，如果在系统表里找到匹配，则将 `i-number` 替换为根目录的 `i-number` ， `file name` 替换为 `table` 中的 `value` 。

### I/O性能小Trick
A program that reads or writes files in units of 512 bytes has an advantage over a program that reads  
or writes a single byte at a time, but the gain is not immense; it comes mainly from the avoidance of system  
overhead. If a program is used rarely or does no great volume of I/O, it may quite reasonably read and  
write in units as small as it wishes.

### i-list分析
结论：quite reliable and easy to deal with
strengths：
- Each file has a short, unambiguous name related in a simple way to the protection, addressing, and  
other information needed to access the file.
- It also permits a quite simple and rapid algorithm for checking  the consistency of a file system：易于验证各个设备上的包含有用信息的部分和可分配的部分是不相交的。（how？）

other question：there is the question of who is to be charged for the space a file occupies, because all  
directory entries for a file have equal status. Charging the owner of a file is unfair in general, for one user  
may create a file, another may link to it, and the first user may delete the file. The first user is still the  
owner of the file, but it should be charged to the second user. The simplest reasonably fair algorithm seems  
to be to spread the charges equally among users who have links to a file. Many installations avoid the issue  
by not charging any fees at all.


# PROCESSES AND IMAGES
## Processes
- fork：copies of the original memory image、share all open files
- pipe：inter-process channel
- execute：All the code and data in the process invoking execute is replaced from the file, but open files, current directory, and inter-process relationships are unaltered.
- wait：causes its caller to suspend execution until one of its children has completed execution
- exit：terminates a process, destroys its image, closes its open files, and generally obliterates it
## Shell

