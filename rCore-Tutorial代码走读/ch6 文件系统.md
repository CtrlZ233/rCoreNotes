# rCore代码笔记-ch6
[[2022-08-13]]

在此之前，我们的操作系统已经可以支持多个应用程序的批量加载，并行执行和进程调度。然后回想一下我们是怎么加载应用程序的：把应用程序的二进制文件和操作系统的二进制文件一起打包，将应用程序的二进制代码作为操作系统的数据段，在操作系统需要的时候，将数据段中的二进制文件拷贝到一个地方进行执行。

试想一下，当我们的应用程序更改的时候，我们不止需要编译应用程序，还需要重新编译链接操作系统；此外，当应用程序的数量和大小膨胀到一定程度后，操作系统的数据段会也会跟着膨胀，这显然不符合我们的预期。

在这章里，我们将应用程序和操作系统彻底分离，将应用程序的镜像放到一个持久化存储介质中，让操作系统在运行时从该介质中读取应用程序数据。而我们的主要工作就是提供对该介质的读写管理接口。

参考资料：[第六章：文件系统 — rCore-Tutorial-Book-v3 3.6.0-alpha.1 文档 (rcore-os.github.io)](http://rcore-os.cn/rCore-Tutorial-Book-v3/chapter6/)

## 文件系统抽象
我们将在磁盘设备上管理文件存储、读写的系统称为文件系统，让操作系统直接操作块设备仍然是极为不便的，此外还涉及到读写效率的问题，因此我们需要对文件系统做几层封装，以便更好地适用于操作系统的使用。

- 磁盘块设备接口层：读写磁盘块设备的trait接口
- 块缓存层：位于内存的磁盘块数据缓存
- 磁盘数据结构层：表示磁盘文件系统的数据结构
- 磁盘块管理器层：实现对磁盘文件系统的管理
- 索引节点层：实现文件创建/文件打开/文件读写等操作

### 磁盘块设备接口层
我们对每一个磁盘块设备都要实现一个Trait，以提供块设备读写的能力：
```rust
// easy-fs/src/block-dev.rs
pub trait BlockDevice: Send + Sync + Any {  
    ///Read data form block to buffer  
    fn read_block(&self, block_id: usize, buf: &mut [u8]);  
    ///Write data from buffer to block  
    fn write_block(&self, block_id: usize, buf: &[u8]);  
}
```
它需要实现两个抽象方法：
-   `read_block` 将编号为 `block_id` 的块从磁盘读入内存中的缓冲区 `buf` ；
-   `write_block` 将内存中的缓冲区 `buf` 中的数据写入磁盘编号为 `block_id` 的块。

以qemu为例，我们在制作一个文件系统盘的时候，使用的是一个宿主机上的一个文件模拟的，因此我们将宿主机上的一个文件作为一个块设备，实现 `BlockDevice` Trait：
```rust
// easy-fs-fuse/src/main.rs
struct BlockFile(Mutex<File>);

impl BlockDevice for BlockFile {
    fn read_block(&self, block_id: usize, buf: &mut [u8]) {
        let mut file = self.0.lock().unwrap();
        file.seek(SeekFrom::Start((block_id * BLOCK_SZ) as u64))
            .expect("Error when seeking!");
        assert_eq!(file.read(buf).unwrap(), BLOCK_SZ, "Not a complete block!");
    }

    fn write_block(&self, block_id: usize, buf: &[u8]) {
        let mut file = self.0.lock().unwrap();
        file.seek(SeekFrom::Start((block_id * BLOCK_SZ) as u64))
            .expect("Error when seeking!");
        assert_eq!(file.write(buf).unwrap(), BLOCK_SZ, "Not a complete block!");
    }
}
```
它将一个文件分成若干个大小为 `BLOCK_SZ` 的块，根据块id来进行读写。

然后在qemu加载操作系统的时候使用参数：
```
-drive file=../user/target/riscv64gc-unknown-none-elf/release/fs.img,if=none,format=raw,id=x0 \
     -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0
```
将fs.img文件作为模拟块设备 `virtio-blk-device` 传入qemu。

而运行在qemu中的操作系统需要为 `virtio-blk-device` 实现 `BlockDevice`  Trait。
```rust
impl BlockDevice for VirtIOBlock {
    fn read_block(&self, block_id: usize, buf: &mut [u8]) {
        self.0
            .exclusive_access()
            .read_block(block_id, buf)
            .expect("Error when reading VirtIOBlk");
    }
    fn write_block(&self, block_id: usize, buf: &[u8]) {
        self.0
            .exclusive_access()
            .write_block(block_id, buf)
            .expect("Error when writing VirtIOBlk");
    }
}
```

它在内层调用了开源库的read和write接口。

### 块缓存层
由于操作系统频繁读写速度缓慢的磁盘块会极大降低系统性能，因此常见的手段是先通过 `read_block` 将一个块上的数据从磁盘读到内存中的一个缓冲区中，这个缓冲区中的内容是可以直接读写的，那么后续对这个数据块的大部分访问就可以在内存中完成了。如果缓冲区中的内容被修改了，那么后续还需要通过 `write_block` 将缓冲区中的内容写回到磁盘块中。

块缓存 `BlockCache` 的定义如下：
```rust
pub struct BlockCache {  
    /// cached block data  
    cache: [u8; BLOCK_SZ],  
    /// underlying block id  
    block_id: usize,  
    /// underlying block device  
    block_device: Arc<dyn BlockDevice>,  
    /// whether the block is dirty  
    modified: bool,  
}
```

它缓存了一个块的数据，并且缓存了块的id和所属块设备，还维护了当前块缓存的更改状态。当我们创建一个 `BlockCache` 时会将对应的块数据读入内存：
```rust
pub fn new(block_id: usize, block_device: Arc<dyn BlockDevice>) -> Self {
	let mut cache = [0u8; BLOCK_SZ];
	block_device.read_block(block_id, &mut cache);
	Self {
		cache,
		block_id,
		block_device,
		modified: false,
	}
}
```

然后我们可以在BlockCache上定义各种读写操作：
```rust
impl BlockCache {
	fn addr_of_offset(&self, offset: usize) -> usize;
	pub fn get_ref<T>(&self, offset: usize) -> &T;
	pub fn get_mut<T>(&mut self, offset: usize) -> &mut T
}
```

- `addr_of_offset` 读取该块中的偏移为 `offset` 的字节。
- `get_ref` 以不可变引用的形式返回偏移量 `offset` 的范型类型 `T`。
- `get_mut` 以可变引用的形式返回偏移量 `offset` 的范型类型 `T`。

```rust
impl BlockCache {
	pub fn read<T, V>(&self, offset: usize, f: impl FnOnce(&T) -> V) -> V;
	pub fn modify<T, V>(&mut self, offset: usize, f: impl FnOnce(&mut T) -> V) -> V;
	pub fn sync(&mut self);
}
```

- `read` 从 `offset` 读取类型为 `T` 的数据并经过一系列不可变操作返回类型 `V` ,不可变操作通过 `f` 传入。
- `modify` 从 `offset` 读取类型为 `T` 的数据并经过一系列写操作返回类型 `V` ,写操作通过 `f` 传入。
- `sync` 检查缓存块中的修改位，若被修改，则写入块设备中。

为了防止我们忘记调用 `sync` 方法回写块设备，我们为 `BlockCache` 实现 `Drop Trait` 。
```rust
impl Drop for BlockCache {
    fn drop(&mut self) {
        self.sync()
    }
}
```

为了避免在块缓存上浪费过多内存，我们希望内存中同时只能驻留有限个磁盘块的缓冲区，同时能有个管理器帮助我们对缓存进行申请和释放：
```rust
// easy-fs/src/block_cache.rs
const BLOCK_CACHE_SIZE: usize = 16;

pub struct BlockCacheManager {
    queue: VecDeque<(usize, Arc<Mutex<BlockCache>>)>,
}
```

然后我们为 `BlockCacheManager` 实现获取块缓存的方法：
```rust
impl BlockCacheManager {
	pub fn get_block_cache(
        &mut self,
        block_id: usize,
        block_device: Arc<dyn BlockDevice>,
    ) -> Arc<Mutex<BlockCache>>
}
```

该方法会判断 传入的 `block_id` 是否在缓存中，如果在则直接返回，如果不在，需要申请新的块缓存，在申请之前，需要判断 `queue` 是否满，满了则需要寻找一个没有被其他引用的缓存块回写删除。

### 磁盘数据结构层
在定义磁盘数据结构之前，我们需要先规定磁盘的布局：
在 easy-fs 磁盘布局中，按照块编号从小到大顺序地分成 5 个不同属性的连续区域：
-   最开始的区域的长度为一个块，其内容是 easy-fs **超级块** (Super Block)。超级块内以魔数的形式提供了文件系统合法性检查功能，同时还可以定位其他连续区域的位置。
-   第二个区域是一个索引节点（inode)位图，长度为若干个块。它记录了后面的索引节点区域中有哪些索引节点已经被分配出去使用了，而哪些还尚未被分配出去。
-   第三个区域是索引节点区域，长度为若干个块。其中的每个块都存储了若干个索引节点。
-   第四个区域是一个数据块位图，长度为若干个块。它记录了后面的数据块区域中有哪些数据块已经被分配出去使用了，而哪些还尚未被分配出去。
-   最后的区域则是数据块区域，顾名思义，其中的每一个已经分配出去的块保存了文件或目录中的具体数据内容。

**索引节点** (Inode, Index Node) 是文件系统中的一种重要数据结构。逻辑目录树结构中的每个文件和目录都对应一个 inode ，我们前面提到的文件系统实现中，文件/目录的底层编号实际上就是指 inode 编号。在 inode 中不仅包含了我们通过 `stat` 工具能够看到的文件/目录的元数据（大小/访问权限/类型等信息），还包含实际保存对应文件/目录数据的数据块（位于最后的数据块区域中）的索引信息，从而能够找到文件/目录的数据被保存在磁盘的哪些块中。从索引方式上看，同时支持直接索引和间接索引。

#### 超级块
超级块 `SuperBlock` 的内容如下：
```rust
// easy-fs/src/layout.rs
#[repr(C)]  
pub struct SuperBlock {  
    magic: u32,  
    pub total_blocks: u32,  
    pub inode_bitmap_blocks: u32,  
    pub inode_area_blocks: u32,  
    pub data_bitmap_blocks: u32,  
    pub data_area_blocks: u32,  
}
```

- `magic` 魔数，为 `0x3b800001`，用于校验超级块是否合法。
- `total_blocks` 文件系统中的总块数。
- `inode_bitmap_blocks`  inode位图所占的块数。
- `inode_area_blocks` inode所占的块数。
- `data_bitmap_blocks` 数据位图所占的块数。
- `data_area_blocks` 数据所占的块数。

#### 位图
这里用于管理各个块的状态，一个bit位表示一个块，它本身也是用若干Block来存储的，因此它的定义如下：
```rust
pub struct Bitmap {  
    start_block_id: usize,  
    blocks: usize,  
}
```
它保存了位图所在的起始块id，和整个位图所占的块数量。
```rust
impl Bitmap {
	pub fn new(start_block_id: usize, blocks: usize) -> Self;
	pub fn alloc(&self, block_device: &Arc<dyn BlockDevice>) -> Option<usize>;
	pub fn dealloc(&self, block_device: &Arc<dyn BlockDevice>, bit: usize);
}
```
然后实现了 `alloc` 和 `dealloc` 方法，用于记录分配和释放的块。

#### DiskInode
磁盘上保存这文件的索引节点，每个索引节点对应一个文件的基本信息，具体定义如下：
```rust
#[repr(C)]  
pub struct DiskInode {  
    pub size: u32,  
    pub direct: [u32; INODE_DIRECT_COUNT],  
    pub indirect1: u32,  
    pub indirect2: u32,  
    type_: DiskInodeType,  
}
```
每个文件/目录在磁盘上均以一个 `DiskInode` 的形式存储：
- `size` 表示文件/目录内容的字节数
- `type_` 表示索引节点的类型 `DiskInodeType` ，目前仅支持文件 `File` 和目录 `Directory` 两种类型。
- 其余的 `direct/indirect1/indirect2` 都是存储文件内容/目录内容的数据块的索引，这也是索引节点名字的由来。

从该数据结构中我们可以看出，一个 `DiskInode` 的大小为为128字节，一级索引可以支持对应的文件的大小不超过 `INODE_DIRECT_COUNT` × `BLOCK_SZ` 时，二级索引可以支持文件大小不超过 （`BLOCK_SZ` / `DiskInodeSize`）× `BLOCK_SZ` 。
三级索引可以支持文件大小不超过：（`BLOCK_SZ` / `DiskInodeSize`）  × （`BLOCK_SZ` / `DiskInodeSize` × `BLOCK_SZ`） 。

然后我们会为 `DiskInode` 实现各种方法：
```rust
impl DiskInode {
	// 索引全部设置为0，设置type
	pub fn initialize(&mut self, type_: DiskInodeType);
	// 判断是否为目录
	pub fn is_dir(&self) -> bool;
	// 判断是否为文件
	pub fn is_file(&self) -> bool;
	// 根据文件的内部相对块序号，获取实际块序号
	pub fn get_block_id(&self, inner_id: u32, block_device: &Arc<dyn BlockDevice>) -> u32;
	// 获取当前文件需要的块数
	pub fn data_blocks(&self) -> u32;
	// 获取size大小需要的块数
	fn _data_blocks(size: u32) -> u32;
	// 获取包括存储间接索引在内，需要的块数
	pub fn total_blocks(size: u32) -> u32;
	// 获取还需要新增多少块数
	pub fn blocks_num_needed(&self, new_size: u32) -> u32;
	// new_size 为扩充之后的文件大小，new_blocks是bitmap分配的块
	pub fn increase_size(
        &mut self,
        new_size: u32,
        new_blocks: Vec<u32>,
        block_device: &Arc<dyn BlockDevice>,
    );
	// 将文件size清0,并返回需要释放的块id的序列
    pub fn clear_size(&mut self, block_device: &Arc<dyn BlockDevice>) -> Vec<u32>;
    // 读取文件偏移为offset的数据到buf
    pub fn read_at(
        &self,
        offset: usize,
        buf: &mut [u8],
        block_device: &Arc<dyn BlockDevice>,
    ) -> usize;
    将buf的数据写入文件偏移为offset处
    pub fn write_at(  
	    &mut self,  
	    offset: usize,  
	    buf: &[u8],  
	    block_device: &Arc<dyn BlockDevice>,  
	) -> usize
}
```

#### 文件和目录项
作为一个文件而言，它的内容在文件系统看来没有任何既定的格式，都只是一个字节序列。因此每个保存内容的数据块都只是一个字节数组。而对于目录而言，需要我们定义一种特定的格式，Inode的type为目录时，索引到的块以我们定义的目录格式读取出来：
```rust
// easy-fs/src/layout.rs

const NAME_LENGTH_LIMIT: usize = 27;

#[repr(C)]
pub struct DirEntry {
    name: [u8; NAME_LENGTH_LIMIT + 1],
    inode_number: u32,
}

pub const DIRENT_SZ: usize = 32;
```

目录项 `DirEntry` 最大允许保存长度为 27 的文件/目录名（数组 `name` 中最末的一个字节留给 `\0` ），还保存了一个目录项对应的Inode编号，且它自身占据空间 32 字节，每个数据块可以存储 16 个目录项。

###  磁盘块管理器层
上一节所介绍的数据结构都是存储在磁盘上，我们定义了能把它们读到内存一系列操作，在这一节，我们就需要在实现在内存中对磁盘写入一个文件系统、打开一个文件系统、操作一个文件系统等。

我们先定义文件系统在内存中操作，所需要维护的数据结构：
```rust
pub struct EasyFileSystem {  
    ///Real device  
    pub block_device: Arc<dyn BlockDevice>,  
    ///Inode bitmap  
    pub inode_bitmap: Bitmap,  
    ///Data bitmap  
    pub data_bitmap: Bitmap,  
    inode_area_start_block: u32,  
    data_area_start_block: u32,  
}
```

然后在此基础上实现了一系列对文件系统操作的方法：
```rust
impl EasyFileSystem {

	// 创建一个文件系统
	pub fn create(
        block_device: Arc<dyn BlockDevice>,
        total_blocks: u32,
        inode_bitmap_blocks: u32,
    ) -> Arc<Mutex<Self>>;

	// 从设备中打开一个文件系统
	pub fn open(block_device: Arc<dyn BlockDevice>) -> Arc<Mutex<Self>>;

	// 获取文件系统中的根节点(disk inode id = 0)
	pub fn root_inode(efs: &Arc<Mutex<Self>>) -> Inode;

	// 获取dis inode在磁盘中的块id和块偏移
	pub fn get_disk_inode_pos(&self, inode_id: u32) -> (u32, usize);

	// 获取数据块的块id
	pub fn get_data_block_id(&self, data_block_id: u32) -> u32;

	// 从inode位图中分配一个inode
	pub fn alloc_inode(&mut self) -> u32;

	// 从data位图中分配一个块，并返回块id
	pub fn alloc_data(&mut self) -> u32;

	// 清空块中的数据，并释放data位图的块id
	pub fn dealloc_data(&mut self, block_id: u32);
	
}
```

我们着重看一下create方法：
```rust
pub fn create(  
    block_device: Arc<dyn BlockDevice>,  
    total_blocks: u32,  
    inode_bitmap_blocks: u32,  
) -> Arc<Mutex<Self>> {  
	...
    let inode_num = inode_bitmap.maximum();  
    let inode_area_blocks =  
        ((inode_num * core::mem::size_of::<DiskInode>() + BLOCK_SZ - 1) / BLOCK_SZ) as u32;  
    let inode_total_blocks = inode_bitmap_blocks + inode_area_blocks;  
    let data_total_blocks = total_blocks - 1 - inode_total_blocks;  
    let data_bitmap_blocks = (data_total_blocks + 4096) / 4097;  
    let data_area_blocks = data_total_blocks - data_bitmap_blocks;  
    for i in 0..total_blocks {  
        get_block_cache(i as usize, Arc::clone(&block_device))  
            .lock()  
            .modify(0, |data_block: &mut DataBlock| {  
                for byte in data_block.iter_mut() {  
                    *byte = 0;  
                }
            });
    }    // initialize SuperBlock  
    get_block_cache(0, Arc::clone(&block_device)).lock().modify(  
        0,  
        |super_block: &mut SuperBlock| {  
            super_block.initialize(  
                total_blocks,  
                inode_bitmap_blocks,  
                inode_area_blocks,  
                data_bitmap_blocks,  
                data_area_blocks,  
            );
       },
    );    // write back immediately  
    ...
}
```

我们之前讲过磁盘的布局，我们文件系统的创建就是严格按照介绍过的磁盘布局来进行的，我们需要先清空所有的块，构造 `SuperBlock`，超级块需要知道的参数有：
- `total_blocks` ，入参已经传入。
- `inode_bitmap_blocks` ，入参已经传入。
- `inode_area_blocks`，通过`inode_bitmap_blocks` 计算出最多有多少个`inode` （`inode_bitmap_blocks`×`BLOCK_SZ`×8） ,然后根据 `DiskInode` 的大小计算出存放所有的 `DiskInode` 需要多少块。
- `data_bitmap_blocks` ：由多少个数据块决定，暂时按照`total_blocks`-`inode_bitmap_blocks`-`inode_area_blocks`-1的数据块个数计算（1为超级块）。
- `inode_area_blocks` ，`data_total_blocks` - `data_bitmap_blocks` 所得。

然后构造构造在内存中停留的文件系统结构体：
```rust
pub fn create(  
    block_device: Arc<dyn BlockDevice>,  
    total_blocks: u32,  
    inode_bitmap_blocks: u32,  
) -> Arc<Mutex<Self>> {  
	let inode_bitmap = Bitmap::new(1, inode_bitmap_blocks as usize);
	...
	let data_bitmap = Bitmap::new(  
	    (1 + inode_bitmap_blocks + inode_area_blocks) as usize,  
	    data_bitmap_blocks as usize,  
	);
	let mut efs = Self {  
	    block_device: Arc::clone(&block_device),  
	    inode_bitmap,  
	    data_bitmap,  
	    inode_area_start_block: 1 + inode_bitmap_blocks,  
	    data_area_start_block: 1 + inode_total_blocks + data_bitmap_blocks,  
	};
	...
}

```
- `inode_bitmap`，从块号1开始，数量为入参。
- `data_bitmap` ，从块号（1 + `inode_bitmap_blocks` +`inode_area_blocks`）开始，长度为 `data_bitmap_blocks`。
- `inode_area_start_block` 和 `data_area_start_block` 都可简单计算出来。

最后创建根目录：
```rust
pub fn create(  
    block_device: Arc<dyn BlockDevice>,  
    total_blocks: u32,  
    inode_bitmap_blocks: u32,  
) -> Arc<Mutex<Self>> {
	...
	assert_eq!(efs.alloc_inode(), 0);  
	let (root_inode_block_id, root_inode_offset) = efs.get_disk_inode_pos(0);  
	get_block_cache(root_inode_block_id as usize, Arc::clone(&block_device))  
	    .lock()  
	    .modify(root_inode_offset, |disk_inode: &mut DiskInode| {  
	        disk_inode.initialize(DiskInodeType::Directory);  
	    });
	block_cache_sync_all();  
	Arc::new(Mutex::new(efs))
}
```
先分配一个 `DiskInode`，这必然是第一个Inode，所以编号为0，然后根据inode编号获取块id和偏移，然后将该处的数据以DiskNode形式读出来并初始化为目录类型的Inode。然后用缓存管理器把所有修改回写到块设备中，完成文件系统在磁盘中的创建。

### 索引节点层
`EasyFileSystem` 实现了磁盘布局并能够将磁盘块有效的管理起来。但是对于文件系统的使用者而言，他们往往不关心磁盘布局是如何实现的，而是更希望能够直接看到目录树结构中逻辑上的文件和目录。为此需要设计索引节点 `Inode` 暴露给文件系统的使用者，让他们能够直接对文件和目录进行操作。 `Inode` 和 `DiskInode` 的区别从它们的名字中就可以看出： `DiskInode` 放在磁盘块中比较固定的位置，而 `Inode` 是放在内存中的记录文件索引节点信息的数据结构。

```rust
// easy-fs/src/vfs.rs
pub struct Inode {
    block_id: usize,
    block_offset: usize,
    fs: Arc<Mutex<EasyFileSystem>>,
    block_device: Arc<dyn BlockDevice>,
}
```

Inode和DiskInode是一一对应的，因此我们用block_id和block_offset来索引DiskInode在磁盘中的位置。
然后为它实现读写DiskNode的方法：
```rust
// easy-fs/src/vfs.rs

impl Inode {
    fn read_disk_inode<V>(&self, f: impl FnOnce(&DiskInode) -> V) -> V {
        get_block_cache(
            self.block_id,
            Arc::clone(&self.block_device)
        ).lock().read(self.block_offset, f)
    }

    fn modify_disk_inode<V>(&self, f: impl FnOnce(&mut DiskInode) -> V) -> V {
        get_block_cache(
            self.block_id,
            Arc::clone(&self.block_device)
        ).lock().modify(self.block_offset, f)
    }
}
```

查找文件：
```rust
impl Inode {
    pub fn find(&self, name: &str) -> Option<Arc<Inode>> {
        let fs = self.fs.lock();
        self.read_disk_inode(|disk_inode| {
            self.find_inode_id(name, disk_inode)
            .map(|inode_id| {
                let (block_id, block_offset) = fs.get_disk_inode_pos(inode_id);
                Arc::new(Self::new(
                    block_id,
                    block_offset,
                    self.fs.clone(),
                    self.block_device.clone(),
                ))
            })
        })
    }
	fn find_inode_id(
		&self,
		name: &str,
		disk_inode: &DiskInode,
	) -> Option<u32>{
		assert!(disk_inode.is_dir());
        let file_count = (disk_inode.size as usize) / DIRENT_SZ;
        let mut dirent = DirEntry::empty();
        for i in 0..file_count {
            assert_eq!(
                disk_inode.read_at(
                    DIRENT_SZ * i,
                    dirent.as_bytes_mut(),
                    &self.block_device,
                ),
                DIRENT_SZ,
            );
            if dirent.name() == name {
                return Some(dirent.inode_number() as u32);
            }
        }
        None
	}
}
```

它先读出当前Inode对应的DiskInode，判断DiskInode是否是目录（只有目录才需要查找文件），然后读取DiskInode对应的块中的所有目录项，返回名字为入参的目录项的inode_id，获取该DiskInode对应的块号和偏移，构造一个新的Inode返回。

文件创建、列举、清空的方法也类似，在此不再赘述。
```rust
impl Inode {
	pub fn create(&self, name: &str) -> Option<Arc<Inode>>;
	pub fn ls(&self) -> Vec<String>;
	pub fn read_at(&self, offset: usize, buf: &mut [u8]) -> usize;
	pub fn write_at(&self, offset: usize, buf: &[u8]) -> usize;
	pub fn clear(&self);
}
```

## 将应用打包到镜像
如今，我们已经完全实现了一个文件系统，现在要做的就是把应用程序打包到文件系统所在的设备中，纳入文件系统的管理：
```rust
// easy-fs-fuse/src/main.rs
fn easy_fs_pack() -> std::io::Result<()> {
	...
	let block_file = Arc::new(BlockFile(Mutex::new({
        let f = OpenOptions::new()
            .read(true)
            .write(true)
            .create(true)
            .open(format!("{}{}", target_path, "fs.img"))?;
        f.set_len(16 * 2048 * 512).unwrap();
        f
    })));
    // 16MiB, at most 4095 files  
	let efs = EasyFileSystem::create(block_file, 16 * 2048, 1);  
	let root_inode = Arc::new(EasyFileSystem::root_inode(&efs));
	...
	for app in apps {  
	    // load app data from host file system  
	    let mut host_file = File::open(format!("{}{}", target_path, app)).unwrap();  
	    let mut all_data: Vec<u8> = Vec::new();  
	    host_file.read_to_end(&mut all_data).unwrap();  
	    // create a file in easy-fs  
	    let inode = root_inode.create(app.as_str()).unwrap();  
	    // write data to easy-fs  
	    inode.write_at(0, all_data.as_slice());  
	}
	...
}
```

我们打开了一个文件并设置大小为16MIB，然后在文件上创建一个文件系统，获取根节点，遍历所有应用程序，将每个应用程序在根目录下创建一个文件进行存放。到此，我们就将所有的应用都打包到了镜像文件中。只需要qemu启动时将镜像文件挂载到外设之后就可以读取了。

## 在操作系统中接入fs

事实上，在文件系统中，除了我们的应用程序之外，还可以拥有一些普通文件。站在用户的角度来看，在一个进程中可以使用多种不同的标志来打开一个文件，这会影响到打开的这个文件可以用何种方式被访问。此外，在连续调用 `sys_read/write` 读写一个文件的时候，我们知道进程中也存在着一个文件读写的当前偏移量，它也随着文件读写的进行而被不断更新。这些要求我们在内核中维护文件的读写权限和偏移状态，因此我们需要在内核中定义一个新的数据结构：
```rust
// os/src/fs/inode.rs

pub struct OSInode {
    readable: bool,
    writable: bool,
    inner: Mutex<OSInodeInner>,
}

pub struct OSInodeInner {
    offset: usize,
    inode: Arc<Inode>,
}

impl OSInode {
    pub fn new(
        readable: bool,
        writable: bool,
        inode: Arc<Inode>,
    ) -> Self {
        Self {
            readable,
            writable,
            inner: Mutex::new(OSInodeInner {
                offset: 0,
                inode,
            }),
        }
    }
}
```

而基于Linux系统一切皆文件的思想，我们定义了一个 `File Trait`（后面涉及到的管道、外设等都可以实现File Trait）
```rust
pub trait File: Send + Sync {  
    /// If readable  
    fn readable(&self) -> bool;  
    /// If writable  
    fn writable(&self) -> bool;  
    /// Read file to `UserBuffer`  
    fn read(&self, buf: UserBuffer) -> usize;  
    /// Write `UserBuffer` to file  
    fn write(&self, buf: UserBuffer) -> usize;  
}
```

然后为OSInode实现该Trait：
```rust
impl File for OSInode {  
    fn readable(&self) -> bool {  
        self.readable  
    }  
    fn writable(&self) -> bool {  
        self.writable  
    }  
    fn read(&self, mut buf: UserBuffer) -> usize {  
        let mut inner = self.inner.exclusive_access();  
        let mut total_read_size = 0usize;  
        for slice in buf.buffers.iter_mut() {  
            let read_size = inner.inode.read_at(inner.offset, *slice);  
            if read_size == 0 {  
                break;  
            }            inner.offset += read_size;  
            total_read_size += read_size;  
        }        
        total_read_size  
    }  
    fn write(&self, buf: UserBuffer) -> usize {  
        let mut inner = self.inner.exclusive_access();  
        let mut total_write_size = 0usize;  
        for slice in buf.buffers.iter() {  
            let write_size = inner.inode.write_at(inner.offset, *slice);  
            assert_eq!(write_size, slice.len());  
            inner.offset += write_size;  
            total_write_size += write_size;  
        }        
        total_write_size  
    }  
}
```

同时，从进程管理的视角看，我打开的文件是属于我当前进程所拥有的资源，因此，进程必须维护好它所使用的文件，我们把进程所使用的文件保存到任务控制块中：
```rust
pub struct TaskControlBlockInner {  
    pub trap_cx_ppn: PhysPageNum,  
    pub base_size: usize,  
    pub task_cx: TaskContext,  
    pub task_status: TaskStatus,  
    pub memory_set: MemorySet,  
    pub parent: Option<Weak<TaskControlBlock>>,  
    pub children: Vec<Arc<TaskControlBlock>>,  
    pub exit_code: i32,  
    pub fd_table: Vec<Option<Arc<dyn File + Send + Sync>>>,  
}
```
当然，我们之前的标准输入、输出和标准错误也都抽象成文件的一部分：
```rust
impl TaskControlBlock {
	pub fn new(elf_data: &[u8]) -> Self {
		...
		let task_control_block = Self {  
		    pid: pid_handle,  
		    kernel_stack,  
		    inner: unsafe {  
		        UPSafeCell::new(TaskControlBlockInner {  
		            trap_cx_ppn,  
		            base_size: user_sp,  
		            task_cx: TaskContext::goto_trap_return(kernel_stack_top),  
		            task_status: TaskStatus::Ready,  
		            memory_set,  
		            parent: None,  
		            children: Vec::new(),  
		            exit_code: 0,  
		            fd_table: vec![  
		                // 0 -> stdin  
		                Some(Arc::new(Stdin)),  
		                // 1 -> stdout  
		                Some(Arc::new(Stdout)),  
		                // 2 -> stderr  
		                Some(Arc::new(Stdout)),  
		            ],
		        })    
		    },
		};
		...
	}
}
```

至此，一个携带文件系统的操作系统就基本完成。