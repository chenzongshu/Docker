BoltDB是一个精干的K/V数据库，大量应用在Go语言相关的程序中

代码量比较少，概念较多，下面来一一解析

# 源码结构

- bucket.go, cursor.go, db.go, freelist.go, node.go, page.go, tx.go
  > 这些文件是BoltDB实现的核心，分别定义和实现了Transaction、Bucket及B+ Tree等机制，也是我们重点阅读与分析的代码。

- bucket_test.go, cursor_test.go,db_test.go, freelist_test.go, node_test.go, page_test.go, quick_test.go, simulation_test.go, tx_test.go
  > 对应的测试文件。

- bolt_linux.go, bolt_openbsd.go, bolt_ppc.go, bolt_unix.go, bolt_unix_solaris.go, bolt_windows.go, boltsync_unix.go
  > 这些文件封装了对应平台下的mmap、fdatasync、flock相关的系统调用。BoltDB只生成一个数据库文件，并通过mmap对文件进行只读映射，写时通过write和fdatasync系统调用修改文件。Go以文件名加"_"，再加"GOOS"或者“GOARCH”的形式或者源文件起始位置添加“// +build”的形式实现不同平台的条件编译。

- bolt_386.go, bolt_amd64.go, bolt_arm.go, bolt_arm64.go, bolt_s390x.go, bolt_ppc.go, bolt_ppc64.go, bolt_ppc64le.go
  > 这些文件定义了对应ABI平台下mmap映射文件大小的上限maxMapSize、分配数组的最大长度maxAllocSize以及是否支持非对齐内存访问(unaligned memory access)。

- errors.go
  > 定义了BoltDB的错误类型及对应的提示字符。

- doc.go
  > 包bolt的说明文档。

# BoltDB使用

```
func main() {
    // Open the database.
    db, err := bolt.Open(“test.db”, 0666, nil)
    if err != nil {
        log.Fatal(err)
    }
    defer os.Remove(db.Path())

    // Start a write transaction.
    if err := db.Update(func(tx *bolt.Tx) error {
        // Create a bucket.
        b, err := tx.CreateBucket([]byte("widgets"))
        if err != nil {
            return err
        }

        // Set the value "bar" for the key "foo".
        if err := b.Put([]byte("foo"), []byte("bar")); err != nil {
            return err
        }
        return nil
    }); err != nil {
        log.Fatal(err)
    }

    // Read value back in a different read-only transaction.
    if err := db.View(func(tx *bolt.Tx) error {
        value := tx.Bucket([]byte("widgets")).Get([]byte("foo"))
        return nil
    }); err != nil {
        log.Fatal(err)
    }

    // Close database to release file lock.
    if err := db.Close(); err != nil {
        log.Fatal(err)
    }
}
```

上面操作有
> 1、调用Open()方法打开或者创建指定文件并得到了DB对象db，
2、然后通过db.Update()方法写数据库，利用Update方法内创建并返回的读写Tx对象创建了一个名为"widgets"的Bucket，并在这个Bucket中添加了一对K/V记录(foo, bar)，
3、随后，通过db.View()方法读数据库，利用View方法内创建并返回的只读Tx对象查找名为widgets的Bucket中的Key为"foo"的记录，
4、最后关闭数据库

# 源码和原理解析

## Open


```
//boltdb/bolt/db.go

func Open(path string, mode os.FileMode, options *Options) (*DB, error) {
    var db = &DB{opened: true}

    ......

    // Open data file and separate sync handler for metadata writes.
    db.path = path
    var err error
    if db.file, err = os.OpenFile(db.path, flag|os.O_CREATE, mode); err != nil {
        _ = db.close()
        return nil, err
    }

    if err := flock(db, mode, !db.readOnly, options.Timeout); err != nil {
        _ = db.close()
        return nil, err
    }

    // Default values for test hooks
    db.ops.writeAt = db.file.WriteAt

    // Initialize the database if it doesn't exist.
    if info, err := db.file.Stat(); err != nil {
        return nil, err
    } else if info.Size() == 0 {
        // Initialize new files with meta pages.
        if err := db.init(); err != nil {
            return nil, err
        }
    } else {
        // Read the first meta page to determine the page size.
        var buf [0x1000]byte
        if _, err := db.file.ReadAt(buf[:], 0); err == nil {
            m := db.pageInBuffer(buf[:], 0).meta()
            if err := m.validate(); err != nil {
                // If we can't read the page size, we can assume it's the same
                // as the OS -- since that's how the page size was chosen in the
                // first place.
                //
                // If the first page is invalid and this OS uses a different
                // page size than what the database was created with then we
                // are out of luck and cannot access the database.
                db.pageSize = os.Getpagesize()
            } else {
                db.pageSize = int(m.pageSize)
            }
        }
    }
    ......
    // Memory map the data file.
    if err := db.mmap(options.InitialMmapSize); err != nil {
        _ = db.close()
        return nil, err
    }
    // Read in the freelist.
    db.freelist = newFreelist()
    db.freelist.read(db.page(db.meta().freelist))

    // Mark the database as opened and return.
    return db, nil
}
```

可以看出，Open()执行的主要步骤为:

> 1、创建DB对象，并将其状态设为opened；
2、打开或创建文件对象
3、根据Open参数ReadOnly决定是否以进程独占的方式打开文件，如果以只读方式访问数据库文件，则不同进程可以共享读该文件；如果以读写方式访问数据库文件，则文件锁将被独占，其他进程无法同时以读写方式访问该数据库文件，这是为了防止多个进程同时修改文件；
4、初始化写文件函数；
5、读数据库文件，如果文件大小为零，则对db进行初始化；如果大小不为零，则试图读取前4K个字节来确定当前数据库的pageSize。随后我们分析db的初始化时，会看到db的文件格式，可以进一步理解这里的逻辑；
6、通过mmap对打开的数据库文件进行内存映射，并初始化db对象中的meta指针；
7、读数据库文件中的freelist页，并初始化db对象中的freelist列表。freelist列表中记录着数据库文件中的空闲页。

这些步骤中比较重要的是第5步中的db.init()调用和第6步中的db.mmap()调用

### db.init()

```
//boltdb/bolt/db.go

// init creates a new database file and initializes its meta pages.
func (db *DB) init() error {
    // Set the page size to the OS page size.
    db.pageSize = os.Getpagesize()

    // Create two meta pages on a buffer.
    buf := make([]byte, db.pageSize*4)
    for i := 0; i < 2; i++ {
        p := db.pageInBuffer(buf[:], pgid(i))
        p.id = pgid(i)
        p.flags = metaPageFlag

        // Initialize the meta page.
        m := p.meta()
        m.magic = magic
        m.version = version
        m.pageSize = uint32(db.pageSize)
        m.freelist = 2
        m.root = bucket{root: 3}
        m.pgid = 4
        m.txid = txid(i)
        m.checksum = m.sum64()
    }

    // Write an empty freelist at page 3.
    p := db.pageInBuffer(buf[:], pgid(2))
    p.id = pgid(2)
    p.flags = freelistPageFlag
    p.count = 0

    // Write an empty leaf page at page 4.
    p = db.pageInBuffer(buf[:], pgid(3))
    p.id = pgid(3)
    p.flags = leafPageFlag
    p.count = 0

    // Write the buffer to our data file.
    if _, err := db.ops.writeAt(buf, 0); err != nil {
        return err
    }
    if err := fdatasync(db); err != nil {
        return err
    }

    return nil
}
```

在init()中:

> 1、先分配了4个page大小的buffer；
2、将第0页和第1页初始化meta页，并指定root bucket的page id为3，存freelist记录的page id为2，当前数据库总页数为4，同时txid分别为0和1。我们将在随后对meta的介绍中说明各个字段的意义；
3、将第2页初始化为freelist页，即freelist的记录将会存在第2页；
4、将第3页初始化为一个空页，它可以用来写入K/V记录，请注意它必须是B+ Tree中的叶子节点；
5、最后，调用写文件函数将buffer中的数据写入文件，同时通过fdatasync()调用将内核中磁盘页缓冲立即写入磁盘。

init()方法创建了一个空的数据库，通过它，我们可以了解boltdb数据库文件的基本格式：数据库文件以页为基本单位，一个数据库文件由若干页组成。

一个页的大小是由当前OS决定的，即通过os.GetpageSize()来决定，对于32位系统，它的值一般为4K字节。

一个Boltdb数据库文件的前两页是meta页，第三页是记录freelist的页面，事实上，经过若干次读写后，freelist页并不一定会存在第三页，也可能不止一页，第4页及后续各页则是用于存储K/V的页面。

init()执行完毕后，新创建的数据库文件大小将是16K字节。随后Open()方法便调用db.mmap()方法对该文件进行映射:

### db.mmap()

```
//boltdb/bolt/db.go

// mmap opens the underlying memory-mapped file and initializes the meta references.
// minsz is the minimum size that the new mmap can be.
func (db *DB) mmap(minsz int) error {
    db.mmaplock.Lock()
    defer db.mmaplock.Unlock()

    info, err := db.file.Stat()
    if err != nil {
        return fmt.Errorf("mmap stat error: %s", err)
    } else if int(info.Size()) < db.pageSize*2 {
        return fmt.Errorf("file size too small")
    }

    // Ensure the size is at least the minimum size.
    var size = int(info.Size())
    if size < minsz {
        size = minsz
    }
    size, err = db.mmapSize(size)
    if err != nil {
        return err
    }

    // Dereference all mmap references before unmapping.
    if db.rwtx != nil {
        db.rwtx.root.dereference()
    }

    // Unmap existing data before continuing.
    if err := db.munmap(); err != nil {
        return err
    }

    // Memory-map the data file as a byte slice.
    if err := mmap(db, size); err != nil {
        return err
    }

    // Save references to the meta pages.
    db.meta0 = db.page(0).meta()
    db.meta1 = db.page(1).meta()

    // Validate the meta pages. We only return an error if both meta pages fail
    // validation, since meta0 failing validation means that it wasn't saved
    // properly -- but we can recover using meta1. And vice-versa.
    err0 := db.meta0.validate()
    err1 := db.meta1.validate()
    if err0 != nil && err1 != nil {
        return err0
    }

    return nil
}
```

在db.mmap()中:

> 1、获取db对象的mmaplock，这里大家可以先忽略它，我们后面再专门介绍DB对象中的锁；
2、通过db.mmapSize()确定mmap映射文件的长度，因为mmap系统调用时要指定映射文件的起始偏移和长度，即确定映射文件的范围；
3、通过munmap()将老的内存映射unmap；
4、通过mmap将文件映射到内存，完成后可以通过db.data来读文件内容了;
5、读数据库文件的第0页和第1页来初始化db.meta0和db.meta1，前面init()方法中我们了解到db的第0面和第1页确实写入的是meta；
6、对meta数据进行校验。

db.mmapSize()的实现比较简单，这里不再贴出其代码，它的思想是：映射文件的最小size为32KB，当文件小于1G时，它的大小以加倍的方式增长，当文件大于1G时，每次remmap增加大小时，是以1G为单位增长的。

前述init()调用完毕后，文件大小是16KB，即db.mmapSize的传入参数是16384，由于mmapSize()限制最小映射文件大小是32768，故它返回的size值为32768，在随后的mmap()调用中第二个传入参数便是32768，即32K。但此时文件大小才16KB，这个时候映射32KB的文件会不会有问题？window平台和linux平台对此有不同的处理:

mmap也是以页为单位进行映射的，如果文件大小不是页大小的整数倍，映射的最后一页肯定超过了文件结尾处，这个时候超过部分的内存会初始化为0，对其的写操作不会写入文件。

但如果映射的内存范围超过了文件大小，且超出范围大于4k，那对于超过文件所在最后一页地址空间的访问将引发异常。比如我们这里文件实际大小是16K，但我们要映射32K到进程地址空间中，那对超过16K部分的内存访问将会引发异常。

实际上，我们前面分析过，Boltdb通过mmap进行了只读映射，故不会存在通过内存映射写文件的问题，同时，对db.data(即映射的内存区域)的访问是通过pgid来访问的，当前database文件里实际包含多少个page是记录在meta中的，每次通过db.data来读取一页时，boltdb均会作超限判断的，所以不会存在对超过当前文件实际页数以外的区域访问的情况。

正如我们在db.init()中看到的，此时meta中记录的pgid为4，即当前数据库文件总的page数为4，故即使mmap映射长度为32KB，通过pgid索引也不会访问到16KB以外的地址空间。

需要说明的是，当对数据库进行写操作时，如果要增加文件大小，针对linux/unix系统，boltdb也会通过ftruncate系统调用增加文件大小，但是它并不是为了避免访问映射区域发生异常的问题，因为boltdb写文件不是通过mmap，而是直接通过fwrite写文件。强调一下，boltdb对数据库的读操作是通过读mmap内存映射区完成的；而写操作是通过文件fseek及fwrite系统调用完成的。

db.mmap()调用完成后，新创建的数据库文件在windows平台上将是32KB，而linux/unix平台仍然是16KB

![](https://upload-images.jianshu.io/upload_images/9246898-ab33fb1d94be0cd0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/529)

## page页格式

```
// boltdb/bolt/page.go

const (
    branchPageFlag   = 0x01
    leafPageFlag     = 0x02
    metaPageFlag     = 0x04
    freelistPageFlag = 0x10
)

......

type pgid uint64

type page struct {
    id       pgid
    flags    uint16
    count    uint16
    overflow uint32
    ptr      uintptr
}
```

> - id: 页面id，如0, 1, 2，...，是从数据库文件内存映射中读取一页的索引值；
- flags: 页面类型，可以分为branchPageFlag、leafPageFlag、metaPageFlag和freelis- tPageFlag等，branchPageFlag和leafPageFlag分别对应B+ Tree 中的内节点和叶子节点， 比如上述看到的第4页(id为3)就被初始化为leafPage；
- count: 页面内存储的元素个数，只在branchPage或leafPage中有用，对应的元素分别为branchPageElement和leafPageElement；
- overflow: 当前页是否有后续页，如果有，overflow表示后续页的数量，如果没有，则它的值为0，主要用于记录连续多页；
- ptr：用于标记页头结尾或者页内存储数据的起始处，一个页的页头就是由上述id、flags、count和overflow构成。需要注意的是，ptr本身不是页头的组成部分，它不是页的一部分，也不被存于磁盘上。

![](https://upload-images.jianshu.io/upload_images/9246898-fee128de7a34b335.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/234)

elements部分的格式根据不同的page类型而不同，我们先看看metaPage的格式，freeListPage的格式比较简单

###  meta page

```
//boltdb/bolt/db.go

type meta struct {
    magic    uint32
    version  uint32
    pageSize uint32
    flags    uint32
    root     bucket
    freelist pgid
    pgid     pgid
    txid     txid
    checksum uint64
}
```

> - magic: boltdb的magic number，为0xED0CDAED;
- version: boltdb文件格式的版本号，为2;
- pageSize：boltdb文件的页大小;
- flags: 保留字段，暂时未用到；
- root: boltdb根Bucket的头信息，后面介绍Bucket时再详细介绍；
- freelist: boltdb文件中存freelist的页号，freelist用来存空闲页面的页号；
- pgid: 简单理解为boltdb文件中总的页数，即最大页号加1。这个字段与BoltDB的MVCC实现有一定联系，我们将在后续文章中再进一步理解它;
- txid: 上一次写数据库的transactoin id， 可以看作是当前boltdb的修改版本号，每次读写数据库时加1，只读时不改变；
- checksum: 上面各字段的64位FNV-1哈希校验。

为了进一步了解meta页是如果存储的，我们可以看看meta的write(*page)方法:

```
//boltdb/bolt/db.go

// write writes the meta onto a page.
func (m *meta) write(p *page) {
    ......

    // Page id is either going to be 0 or 1 which we can determine by the transaction ID.
    p.id = pgid(m.txid % 2)
    p.flags |= metaPageFlag

    // Calculate the checksum.
    m.checksum = m.sum64()

    m.copy(p.meta())
}
```

从上面的代码中可看到，meta信息在写入页框时，

1、指定写入的页号为m.txid % 2、即第0或者第1页，这与我们之前看到的第0页或者第1页初始化为meta页是相符的。更重要的是，写入第0页还是第1页是由当前meta中的transction id决定的， 若当前meta中的transaction id为偶数则写入第0页，若当前meta页的 transaction id为奇数则写入第1页。前面介绍说meta中的txid实际上可以看作是数据库的修改版本号，每次写时会增加1，也就是说每次写数据库后会交替更新meta页。如当前txid为10，它对应的meta存在第0页，当对数据库进行一次读写时，txid增加为11，写完后需要更新meta页，这时会将新的meta写入第1页，而不是覆盖原来的第0页，下次读写数据库时将会选择txid更大的meta页来提取meta信息。我们后面介绍读写数据库时会进一步介绍，这里先提一个问题供大家思考: boltdb为什么要维护两页meta(让我们称之为双meta)呢？
将页面flags设定为metaPageFlag，指明为一个meta页面;
3、将meta信息拷贝到页面p中缓存的相应位置，我们来看看这个位置是如何确定的:

```
//boltdb/bolt/page.go

// meta returns a pointer to the metadata section of the page.
func (p *page) meta() *meta {
    return (*meta)(unsafe.Pointer(&p.ptr))
}
```

可以看到，meta信息被写入了页框的ptr处，即页框头的结尾处。了解了meta的详细信息及写入页框的机制后，我们就可以知道一个meta页面的磁盘布局情况了：

![](https://upload-images.jianshu.io/upload_images/9246898-92e4e9d9d5b4a389.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/210)

到此， 我们就了解了boltdb数据库文件的创建过程及其文件格式。boltdb是以页为单位存储的，并包含起始的两个meta页，一个(或者多个)页来存空闲页的页号及剩下的分支页(branchPage)和叶子页(leafPage)。在创建一个boltdb文件时，通过fwrite和fdatasync系统调用向磁盘写入32K或者16K的初始文件；在打开一个botldb文件时，用mmap系统调用创建只读内存映射，用来读取文件中各页数据。

### branch page

branch page中存的是B+Tree上内节点中的数据，即branchPageElements

```
//boltdb/bolt/page.go

// branchPageElement represents a node on a branch page.
type branchPageElement struct {
    pos   uint32
    ksize uint32
    pgid  pgid
}
```

- pos: element对应的K/V对存储位置相对于当前element的偏移，随后我们会更清楚地理解它;
- ksize：element对应的Key的长度，以字节为单位;
- pgid: element指向的子节点所在page的页号。



### leaf page

leaf page中存的是B+Tree上叶子节点中的数据，即leafPageElements，是存储实际K/V对的地方

B+Tree与B-Tree的一个显著区别便是如此，即B+Tree中只有叶子节点存储实际的K/V对，内节点实际上只用于索引，并且兄弟节点之间会形成有序链表，以加快查找过程；而B-Tree中的K/V是分布在所有节点上的

- flags: 标明当前element是否代表一个Bucket，如果是Bucket则其值为1，如果不是则其值为0;
- pos: 与branchPageElement中的pos字段意义一样;
- ksize: element对应的Key的长度，以字节为单位;
- vsize: element对应的Vaule的长度，以字节为单位。

### 页面布局

一个branchPage或leafPage由页头和若干branchPageElements或leafPageElements组成，那么这些元素是如何在磁盘上布局的呢？我们可以看看node的write(p *page)方法:

```
//boltdb/bolt/node.go

// write writes the items onto one or more pages.
func (n *node) write(p *page) {
    // Initialize page.
    if n.isLeaf {
        p.flags |= leafPageFlag
    } else {
        p.flags |= branchPageFlag
    }

    ......

    // Loop over each item and write it to the page.
    b := (*[maxAllocSize]byte)(unsafe.Pointer(&p.ptr))[n.pageElementSize()*len(n.inodes):]    (1)
    for i, item := range n.inodes {
        _assert(len(item.key) > 0, "write: zero-length inode key")

        // Write the page element.
        if n.isLeaf {
            elem := p.leafPageElement(uint16(i))    (2)
            elem.pos = uint32(uintptr(unsafe.Pointer(&b[0])) - uintptr(unsafe.Pointer(elem)))  (3)
            elem.flags = item.flags
            elem.ksize = uint32(len(item.key))
            elem.vsize = uint32(len(item.value))
        } else {
            elem := p.branchPageElement(uint16(i))      (4)
            elem.pos = uint32(uintptr(unsafe.Pointer(&b[0])) - uintptr(unsafe.Pointer(elem)))     (5)
            elem.ksize = uint32(len(item.key))
            elem.pgid = item.pgid
            _assert(elem.pgid != p.id, "write: circular dependency occurred")
        }

        ......

        // Write data for the element to the end of the page.
        copy(b[0:], item.key)       (6)
        b = b[klen:]                (7)
        copy(b[0:], item.value)     (8)
        b = b[vlen:]                (9)
    }
    ......
}
```

node是一个page加载到内存中后的结构化表示，即它是page反序列化(或者实例化)的结果；相反地，page是node的序列化结果，上述的write()方法就是node的序列化过程。page中存的K/V对反序列化后会存在node.inodes中，page中的elements个数与node.inodes中的个数相同，而且一一对应。
在write(p *page)方法中:

1、根据node是否是叶子节点确定page是leafPage还是branchPage;
2、代码(1)处定义了byte数组指针b，它指向p.ptr指向的位置向后偏移n.pageElementSize()*len(n.inodes)的位置。上文中我们介绍过p.ptr实际上指向页头结尾处或者页正文开始处，所以b实际上是指向了页正文中elements的结尾处，而且elements是从页正文起始处开始存的，这一点我们在下面可以看到；
3、通过for循环将所有K/V记录顺序写入页框;
4、在代码(2)处，定义了一个leafPageElement指针elem，它指向page中第i个leafPageElement:

```
//boltdb/bolt/page.go

// leafPageElement retrieves the leaf node by index
func (p *page) leafPageElement(index uint16) *leafPageElement {
    n := &((*[0x7FFFFFF]leafPageElement)(unsafe.Pointer(&p.ptr)))[index]
    return n
}
```
从上面代码可以看到，当index=0时，第1个leafPageElement是从p.ptr指向的位置开始存的，而且所有leafPageElement是连续存储的，可以通过index进行数组索引。类似地，代码(5)处，定义了一个branchPageLement指针，它指向page中的第i个branchPageElement。在branchPage中，branchPageElement也是从页正文起始处顺序存储的:

```
//boltdb/bolt/page.go

// branchPageElement retrieves the branch node by index
func (p *page) branchPageElement(index uint16) *branchPageElement {
    return &((*[0x7FFFFFF]branchPageElement)(unsafe.Pointer(&p.ptr)))[index]
}
```

5、代码(3)处给leafPageElement的pos字段赋值，它的值为b[0]到当前leafPageElement的偏移，而b[0]实际上位于第i个K/V对的起始位置。当i=0时，b[0]实际上就是页中最后一个element的结尾处。一次for循环，实际上就是写入一个element和一个K/V对。代码(4)处和代码(3)处类似，branchPageElement中的pos字段赋值为b[0]到当前branchPageElement的偏移;
6、代码(6)处将Key写入&b[0]处;
7、代码(7)处将b向前移klen字节，即移到Key的结尾处。习惯于C/C++中指针操作的读者可能对这里b的处理有些疑惑，实际上Go虽然保留了指针，但还是谨慎地禁止了指针的偏移操作，所以试图用指针偏移进行读写的地方均用slice来操作;
8、代码(8)将Value写入&b[0]处，即Value写入Key后的位置;
9、代码(9)将b向前移vlen字节，即移动到Value的结尾处，准备写入下一个Key。

从代码(6)、(7)、(8)和(9)处，我们可以知道，页中存储K/V是以Key、Value交替连续存储的；代码(1)、(3)和(5)可以看到所有的K/V对是在页中的elements结构结尾处存储，且每个element通过pos指向自己对应的K/V对，pos的值即是K/V对相对element起始位置的偏移；而代码(2)和(4)处可以看出，elements是从页正文开始处存储的。需要说明的是，B+Tree中的内结点并不存真正的K/V对，它只存Key用于查找，故branch page中实际上没有存Key的Value，所以branchPageElement中并没像leafPageElement那样定义了vsize，因为branchPageElement的vsize始终是0。

![](https://upload-images.jianshu.io/upload_images/9246898-0a761d47189dae73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/604)

由于OS从磁盘读文件或者内核将页缓冲写入磁盘也是以页大小为单位操作的，BoltDB中每一页的大小与操作系统的页大小保持一致，且也以页为单位对数据库文件进行读写，可以降低磁盘I/O的次数，提高数据库的读写效率。

## 页的组织

BoltDB采用B+Tree来进行查找，page实例化成node，node便是B+Tree中的节点，若干node形成一颗B+Tree。

在BoltDB中，一个Bucket对应一颗B+Tree。那么，对BoltDB的查找，实际上就是对B+Tree的查找；对BoltDB进行K/V读写，就是对B+Tree上的叶子节点读写及对B+Tree进行动态调整。

### transaction

从db.Update()方法来入手分析

```
//boltdb/bolt/db.go

// Update executes a function within the context of a read-write managed transaction.
// If no error is returned from the function then the transaction is committed.
// If an error is returned then the entire transaction is rolled back.
// Any error that is returned from the function or returned from the commit is
// returned from the Update() method.
//
// Attempting to manually commit or rollback within the function will cause a panic.
func (db *DB) Update(fn func(*Tx) error) error {
    t, err := db.Begin(true)
    if err != nil {
        return err
    }

    // Make sure the transaction rolls back in the event of a panic.
    defer func() {
        if t.db != nil {
            t.rollback()
        }
    }()

    // Mark as a managed tx so that the inner function cannot manually commit.
    t.managed = true

    // If an error is returned from the function then rollback and return error.
    err = fn(t)
    t.managed = false
    if err != nil {
        _ = t.Rollback()
        return err
    }

    return t.Commit()
}
```

db.Update(fn func(*Tx) error)的传入参数是一个函数值(function-value)， 且函数的传出参数为一个Tx指针。传入的函数值可以理解为回调函数的函数指针，可以通过函数名(fn)进行调用，通过回调函数传出的Tx指针指向内部创建的一个Transation，调用者通过指向的Transaction对象进行创建、查找、删除及遍历Bucket的操作。db.Update()方法主要执行:

1、通过db.Begin(writeable bool)创建一个Transaction;
2、回调传入的回调函数，将刚创建的Tx指针传出;
3、调用t.Commit()将对BoltDB的修改提交并写入磁盘;
4、如果回调函数返回error，或者transaction在Commit的时候发生异常或错误，当前transation进行的写操作将会回滚(Rollback)，不会写入磁盘。

db.Begin(writeable bool)的代码比较简单，如果传入的writeable为true，则调用db.beginRWTx()创建一个可以读写的transaction；如果writeable为false，则调用db.beginTx()创建一个只读transaction。可以看到，db.Update()会创建可读写transaction，而db.View()将创建只读transaction。

在db.beginRWTx()中:

```
//boltdb/bolt/db.go

func (db *DB) beginRWTx() (*Tx, error) {
    ....

    // Obtain writer lock. This is released by the transaction when it closes.
    // This enforces only one writer transaction at a time.
    db.rwlock.Lock()

    // Once we have the writer lock then we can lock the meta pages so that
    // we can set up the transaction.
    db.metalock.Lock()
    defer db.metalock.Unlock()

    ......

    // Create a transaction associated with the database.
    t := &Tx{writable: true}                                                      (1)
    t.init(db)                                                                    (2)
    db.rwtx = t                                                                   (3)

    ......

    return t, nil
}
```

1、获取读写锁，db.rwlock只有在transaction 被Commit或者Rollback的时候释放， 也即它将锁定读写transaction的整个生命周期，实现了一个进程内同时只有一个读写transaction。请注意，虽然它的名字叫rwlock，但它并不是读写锁，而是一个互斥锁(sync.Mutex)。结合我们前面分析的，调用bolt.Open()方法打开数据库文件时，如果以读写的方式打开，文件锁将会被独占，防止同时有多个进程写文件。结合文件锁与db.rwlock，BoltDB可以保证同一时段只有一个进程的一个线程可以对数据库修改。如果在Go中调用，可以认为只有一个goroutine会修改数据库，尽管一个goroutine可能会被调度到不同的内核线程上。
2、获取db.metalock，需要注意的是，metalock实际上是对db对象的访问保护，特别是对db.txs的读写保护，而不是如名字或者注释中说的专门对meta page的读写保护。
3、代码(1)处新建了一个Tx对象，并通过t来引用，随后代码(2)处调用t.init()方法来对刚刚创建的读写Tx进行初始化;
4、代码(3)处将db.rwtx设为刚刚创建并初始化的transaction。db.txs字段用来记录所有的已打开的只读transaction，它是一个map，从这里也可以看出，BoltDB同时只能有一个可读写transaction，但可以有多个只读transactions;

作为比较，我们再来看看db.beginTx()的实现:

```
//boltdb/bolt/db.go

func (db *DB) beginTx() (*Tx, error) {
    // Lock the meta pages while we initialize the transaction. We obtain
    // the meta lock before the mmap lock because that's the order that the
    // write transaction will obtain them.
    db.metalock.Lock()

    // Obtain a read-only lock on the mmap. When the mmap is remapped it will
    // obtain a write lock so all transactions must finish before it can be
    // remapped.
    db.mmaplock.RLock()

    ......
    // Create a transaction associated with the database.
    t := &Tx{}                      (1)        
    t.init(db)                      (2)

    // Keep track of transaction until it closes.
    db.txs = append(db.txs, t)      (3)
    n := len(db.txs)

    // Unlock the meta pages.
    db.metalock.Unlock()
    ......

    return t, nil
}
```

1、获取db.metalock，因为后面要对db对象进行读写;
2、获取db.mmaplock读锁，db.mmaplock是一个读写锁(sync.RWMutex)，它的读锁在只读transaction关闭的时候释放，也即db.mmaplock的读锁在整个只读transaction的生命周期中被占用。前面我们在db.mmap()中接触过它，db.mmap()的整个过程被db.mmaplock的写锁保护。db.mmap()在两种情况下会被调用: 第一情形便是我们前文介绍的数据文件创建或打开后进行第一次内存映射时；第二种情形是我们后面将要介绍到的，在写入数据库后且数据库文件要增大时，分配新的页框后，需要重新进行mmap系统调用将新的文件范围映射入进程地址空间。在前一种情况中，还没有开始db.beginTx()的调用，故不存在db.mmaplock锁争用问题，但当数据库在不同线程中进行读写时，可能存在其中一个线程中的读写transaction写入了大量数据，在Commit时，由于当前已映射区的空闲页不够，会调用db.mmap()重新进行内存映射，此时若有未关闭的只读transaction，由于它占用着在db.mmaplock的读锁，db.mmap()会阻塞在争用db.mmaplock写锁的地方。也就是说，如果存在着耗时的只读transaction，同时写transaction需要remmap时，写操作会被读操作阻塞。由此可以看出，使用BoltDB时，应尽量避免耗时的读操作，同时在写操作时应避免频繁地remmap，我们将在介绍BoltDB的MVCC机制时再讨论这个问题。
3、代码(1)处创建一个只读transaction对象，代码(2)处对其进行初始化;
4、代码(3)处将刚创建并初始化的只读transation加入到db.txs中。

db.beginRWTx()和db.beginTx()的主要工作均是准备Tx，我们来看一看它的定义:

```
//boltdb/bolt/tx.go

type Tx struct {
    writable       bool
    managed        bool
    db             *DB
    meta           *meta
    root           Bucket
    pages          map[pgid]*page
    stats          TxStats
    commitHandlers []func()

    WriteFlag int
}
```

> - writable: 指示是否是可读写的transaction;
- managed: 指示当前transaction是否被db托管，即通过db.Update()或者db.View()来写或者读数据库。BoltDB还支持直接调用Tx的相关方法进行读写，这时managed字段为false;
- db: 指向当前db对象
- meta: transaction初始化时从db中数到的meta信息;
- root: transaction的根Bucket，所有的transaction均从根Bucket开始进行查找;
- pages: 当前transaction读或写的page;
- stats: 与transaction操作统计相关，不作介绍;
- commitHandlers: transaction在Commit时的回调函数;
- WriteFlag: 复制或移动数据库文件时，指定的文件打开模式，暂不作介绍，有兴趣的读者可以自行分析。

其中比较关键的字段是meta和root，我们将在tx.init()方法中看到它们的初始化过程:

```
//boltdb/bolt/tx.go

// init initializes the transaction.
func (tx *Tx) init(db *DB) {
    tx.db = db
    tx.pages = nil

    // Copy the meta page since it can be changed by the writer.
    tx.meta = &meta{}
    db.meta().copy(tx.meta)

    // Copy over the root bucket.
    tx.root = newBucket(tx)
    tx.root.bucket = &bucket{}
    *tx.root.bucket = tx.meta.root

    // Increment the transaction id and add a page cache for writable transactions.
    if tx.writable {
        tx.pages = make(map[pgid]*page)
        tx.meta.txid += txid(1)
    }
}
```

1、将tx.db初始化为传入的db，将tx.pages初始化为空;
2、创建一个空的meta对象，并用它初始化tx.meta，然后将db中的meta复制到刚创建的meta对象中，请注意，这里的复制是对象拷贝而不是指针拷贝。前面我们介绍过，db中有两个meta，那这里拷贝的是哪一个meta呢？我们来看看db.meta()的代码:

```
//boltdb/bolt/db.go

// meta retrieves the current meta page reference.
func (db *DB) meta() *meta {
    // We have to return the meta with the highest txid which doesn't fail
    // validation. Otherwise, we can cause errors when in fact the database is
    // in a consistent state. metaA is the one with the higher txid.
    metaA := db.meta0
    metaB := db.meta1
    if db.meta1.txid > db.meta0.txid {
        metaA = db.meta1
        metaB = db.meta0
    }

    // Use higher meta page if valid. Otherwise fallback to previous, if valid.
    if err := metaA.validate(); err == nil {
        return metaA
    } else if err := metaB.validate(); err == nil {
        return metaB
    }

    // This should never be reached, because both meta1 and meta0 were validated
    // on mmap() and we do fsync() on every write.
    panic("bolt.DB.meta(): invalid meta pages")
}
```

3、可以看出，db.meta()返回的是两个meta中txid更大且通过校验的那个，前面我们说meta中的txid可以看作是数据库的修改版本号，所以db.meta()返回的meta对应的是数据库最新的状态。

4、通过newBucket(tx *Tx)创建一个Bucket，并将其设为根Bucket，同时用meta中保存的根Bucket的头部来初始化transaction的根Bucket头部。在没有介绍Bucket之前，这里稍微有点不好理解，简单地说，Bucket包括头部(bucket)和一些正文字段，头部中包括了Bucket的根节点所在的页的页号和一个序列号，所以tx.init()中对tx.root的初始化其实主要就是将meta中存的根Bucket(它也是整个db的根Bucket)的头部(bucket)拷贝给当前transaction的根Bucket;
如果是可读写的transaction，就将meta中的txid加1，当可读写transaction commit后，meta就会更新到数据库文件中，数据库的修改版本号就增加了。

**Transaction主要是对Bucket进行创建、查找、删除等操作，Bucket可以嵌套形成树结构，故Transaction对Bucket的操作均从根Bucket开始进行的**

### Bucket

BoltDB中所有的K/V记录或者page均通过Bucket组织起来，且一个Bucket内的所有node形成一颗B+Tree

```
//boltdb/bolt/bucket.go

// Bucket represents a collection of key/value pairs inside the database.
type Bucket struct {
    *bucket
    tx       *Tx                // the associated transaction
    buckets  map[string]*Bucket // subbucket cache
    page     *page              // inline page reference
    rootNode *node              // materialized node for the root page.
    nodes    map[pgid]*node     // node cache

    FillPercent float64
}

type bucket struct {
    root     pgid   // page id of the bucket's root-level page
    sequence uint64 // monotonically incrementing, used by NextSequence()
}
```

> - *bucket: Bucket的头，其定义中的root是指Bucket根节点的页号，sequence是一个序列号，可以用于Bucket的序号生成器。这里稍微解释一下，Go中类型定义中未指明成员名称，且只通过成员类型加星号声明的成员是一个隐含成员，成员的成员以及与其绑定的方法可以被外层类型的变量直接访问或者调用，相当于通过组合的方式让外层内型"继承"了隐含成员的类型;
- tx: 当前Bucket所属的Transaction。请注意，Transaction与Bucket均是动态的概念，即内存中的对象，Bucket最终表示为BoltDB中的K/V记录，这将在后面看到;
- page: 内置(inline)Bucket的页引用，内置Bucket只有一个节点，即它的根节点，且节点不存在独立的页面中，而是作为Bucket的Value存在父Bucket所在页上，页引用就是指向Value中的内置页，我们将在分析Bucket创建时进一步说明;
- rootNode: Bucket的根节点，也是对应B+Tree的根节点;
- nodes: Bucket中的node集合;
- FillPercent: Bucket中节点的填充百分比(或者占空比)。该值与B+Tree中节点的分裂有关系，当节点中Key的个数或者size超过整个node容量的某个百分比后，节点必须分裂为两个节点，这是为了防止B+Tree中插入K/V时引发频繁的再平衡操作，所以注释中提到只有当确定大数多写入操作是向尾添加时，这个值调大才有帮助。该值的默认值是50%;

BoltDB整体结构画像：

![](https://upload-images.jianshu.io/upload_images/9246898-280e4b9b982c2f39.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

图中展示了5个Bucket: root Bucket、BucketA、BucketB、BucketAA及BucketBB。除root Bucket外，其余4个Bucket均由粉色区域标注。

BucketA与BucketB是root Bucket的子Bucket，它们嵌套在root Bucket中; 

BucketAA是BucketA的嵌套子Bucket，BucketBB是BucketB的嵌套子Bucket，而且BucketBB是一个内置Bucket。嵌套Bucket形成父子关系，从而所有Bucket形成树结构，通过根Bucket可以遍历所有子Bucket，

但是请注意，Bucket之间的树结构并不是B+Tree，而是一个逻辑树结构，如BucketBB是BucketB的子Bucket，但并不是说BucketBB所在的节点就是BucketB所在节点的子节点

在BoltDB中，内节点或根节点中Pointer的数量等于Key的数量，即子节点的个数与Key的数量相等，节点中Key的数量的最大值和最小值是根据页大小与Key的size来动态决定的

从图中可以看出，实际的K/V对均保存在叶子节点中，节点中的K/V对按Key的顺序有序排列。父节点中只有Key和Pointer，没有Value，Pointer指向子节点，且Pointer对应的Key是它指向的子节点中最小Key值。父节点中的Key也按顺序排列，所以其子节点之间也是按顺序排列的，所有子节点会形成有序链表。图中的B+Tree上，当我们要查找Key为6的记录时，先从左至右查找(或者通过二分法查找)根节点的Key值，发现6大于4且小于7，那么可以肯定Key为6的记录位于Key为4对应的子节点上，因此进入对应的子节点查找，按相同的查找方法(从左至右或者二分法查找)查找Key为6的记录，最多经过三次比较后就可以找到。同时，在B+Tree中，一个节点中的Key的数量如果大于节点Key数量的最大值 x 填充率的话，节点会分裂(split)成两个节点，并向父节点添加一个Key和Pointer，添加后如果父节点也需要分裂的话那就进行分裂直到根节点；为了尽量减少节点分裂的情况，可以对B+Tree进行旋转(rotation)或者再平衡(rebalance)：如果一个节点中的Key的数量小于设定的节点Key数量的最小值的话，那就将其兄弟节点中的Key移动到该节点，移动后产生的空节点将被删除，直到所有节点满足Key数量大于节点Key数量最小值。

下面从tx.CreatBucket()入手来介绍Bucket的创建过程

```
//boltdb/bolt/tx.go

func (tx *Tx) CreateBucket(name []byte) (*Bucket, error) {
    return tx.root.CreateBucket(name)
}


// CreateBucket creates a new bucket at the given key and returns the new bucket.
// Returns an error if the key already exists, if the bucket name is blank, or if the bucket name is too long.
// The bucket instance is only valid for the lifetime of the transaction.
func (b *Bucket) CreateBucket(key []byte) (*Bucket, error) {

    ......

    // Move cursor to correct position.
    c := b.Cursor()                                                     (1)
    k, _, flags := c.seek(key)                                          (2)

    ......
    // Create empty, inline bucket.
    var bucket = Bucket{                                                (3)
        bucket:      &bucket{},
        rootNode:    &node{isLeaf: true},
        FillPercent: DefaultFillPercent,
    }
    var value = bucket.write()                                          (4)

    // Insert into node.
    key = cloneBytes(key)                                               (5)
    c.node().put(key, key, value, 0, bucketLeafFlag)                    (6)

    // Since subbuckets are not allowed on inline buckets, we need to
    // dereference the inline page, if it exists. This will cause the bucket
    // to be treated as a regular, non-inline bucket for the rest of the tx.
    b.page = nil                                                        (7)

    return b.Bucket(key), nil                                           (8)
}
```

1、代码(1)处为当前Bucket创建游标;
2、代码(2)处在当前Bucket中查找key并移动游标，确定其应该插入的位置;
3、代码(3)处创建一个空的内置Bucket;
4、代码(4)处将刚创建的Bucket序列化成byte slice，以作为与key对应的value插入B+Tree；
5、代码(5)处对key进行深度拷贝以防它引用的byte slice被回收;
6、代码(6)处将代表子Bucket的K/V插入到游标位置，其中Key是子Bucket的名字，Value是子Bucket的序列化结果;
7、代码(7)处将当前Bucket的page字段置空，因为当前Bucket包含了刚创建的子Bucket，它不会是内置Bucket;
8、代码(8)处通过b.Bucket()方法按子Bucket的名字查找子Bucket并返回结果。请注意，这里并没有将刚创建的子Bucket直接返回而是通过查找的方式返回，大家可以想一想为什么？

上述代码涉及到了Bucket的游标Cursor，为了理解Bucket中的B+Tree的查找过程，我们先来看一看Cursor的定义:

```
//boltdb/bolt/cursor.go

type Cursor struct {
    bucket *Bucket
    stack  []elemRef
}

......

// elemRef represents a reference to an element on a given page/node.
type elemRef struct {
    page  *page
    node  *node
    index int
}
```

> Cursor的各字段意义如下:

>- bucket: 要搜索的Bucket的引用;
- stack: 它是一个slice，其中的元素为elemRef，用于记录游标的搜索路径，最后一个元素指向游标当前位置;

> elemRef的各字段意义是:

> - page: elemRef所代表的page的引用;
- node：elemRef所代码的node的引用;
- index：page或node中元素的索引;

elemRef实际上指向B+Tree上的一个节点，节点有可能已经实例化成node，也有可能是未实例化的page。我们前面提到过，Bucket是内存中的动态对象，它缓存了node。Cursor在遍历B+Tree时，如果节点已经实例化成node，则elemRef中的node字段指向该node，page字段为空；如果节点是未实例化的page，则elemRef中的page字段指向该page，node字段为空。elemRef通过page或者node指向B+Tree的一个节点，通过index进一步指向节点中的K/V对。Cursor在B+Tree上的移动过程就是将elemRef添加或者移除出stack的过程。

看看node的定义

```
//boltdb/bolt/node.go

// node represents an in-memory, deserialized page.
type node struct {
    bucket     *Bucket
    isLeaf     bool
    unbalanced bool
    spilled    bool
    key        []byte
    pgid       pgid
    parent     *node
    children   nodes
    inodes     inodes
}

......

// inode represents an internal node inside of a node.
// It can be used to point to elements in a page or point
// to an element which hasn't been added to a page yet.
type inode struct {
    flags uint32
    pgid  pgid
    key   []byte
    value []byte
}

type inodes []inode
```

> node中各字段意义如下:

> - bucket：指向node所属的Bucket;
- isLeaf: 当前node是否是一个叶子节点，与page中的flags对应;
- unbalanced: 指示当前node是否需要再平衡，如果unbalanced为true，则需要对它进行重平衡；当节点中有K/V对被删除时，该值设为true;
- spilled: 指示当前node是否已经溢出(spill)，node溢出是指当node的size超过页大小，即一页无法存node中所有K/V对时，节点会分裂成多个node，以保证每个node的size小于页大小，分裂后产生新的node会增加父node的size，故父node也可能需要分裂；已经溢出过的node不需要再分裂;
- key: node的key或名字，它是node中第1个K/V对的key;
- pgid: 与node对应的页框的页号;
- parent: 父节点的引用，根节点的parent是空;
- children: 当前节点的孩子节点;
- inodes: 一个inode的slice，记录当前node中的K/V;

> inode中各字段的意义是:

> - flags: 指明该K/V对是否代表一个Bucket，如果是Bucket，则其值为1，否则为0;
- pgid: 根节点或内节点的子节点的pgid，可以理解为Pointer，请注意，叶子节点中该值无意义;
- key：K/V对的Key;
- value: K/V对的Value，请注意，非叶子节点中该值无意义;

为了进一步理解node各字段及其与page的对应关系，我们来看看它的反序列化方法read(p *page)：

```
//boltdb/bolt/node.go

// read initializes the node from a page.
func (n *node) read(p *page) {
    n.pgid = p.id
    n.isLeaf = ((p.flags & leafPageFlag) != 0)
    n.inodes = make(inodes, int(p.count))

    for i := 0; i < int(p.count); i++ {
        inode := &n.inodes[i]
        if n.isLeaf {
            elem := p.leafPageElement(uint16(i))
            inode.flags = elem.flags
            inode.key = elem.key()
            inode.value = elem.value()
        } else {
            elem := p.branchPageElement(uint16(i))
            inode.pgid = elem.pgid
            inode.key = elem.key()
        }
        _assert(len(inode.key) > 0, "read: zero-length inode key")
    }

    // Save first key so we can find the node in the parent when we spill.
    if len(n.inodes) > 0 {
        n.key = n.inodes[0].key
        _assert(len(n.key) > 0, "read: zero-length node key")
    } else {
        n.key = nil
    }
}
```

node的实例化过程为:

1、node的pgid设为page的pgid;
2、node的isLeaf字段由page的flags决定，如果page是一个leaf page，则node的isLeaf为true;
创建inodes，其容量由page中元素个数决定;
3、接着，向inodes中填充inode。对于leaf page， inode的flags即为page中元素的flags， key和value分别为page中元素对应的Key和Value；对于branch page， inode的pgid即为page中元素的pgid，即子节点的页号，key为page元素对应的Key。node中的inode与page中的leafPageElement或branchPageElement一一对应，可以参考前文中的页磁盘布局图来理解inode与elemements的对应关系;
4、最后，将node的key设为其中第一个inode的key;

了解了Cursor及node的定义后，我们接着来看看CreateBucket()代码(2)处查找过程:

```
//boltdb/bolt/cursor.go

func (c *Cursor) seek(seek []byte) (key []byte, value []byte, flags uint32) {
    _assert(c.bucket.tx.db != nil, "tx closed")

    // Start from root page/node and traverse to correct page.
    c.stack = c.stack[:0]
    c.search(seek, c.bucket.root)
    ref := &c.stack[len(c.stack)-1]

    // If the cursor is pointing to the end of page/node then return nil.
    if ref.index >= ref.count() {
        return nil, nil, 0
    }

    // If this is a bucket then return a nil value.
    return c.keyValue()
}
```

主要是两个步骤:

1、通过清空stack，将游标复位;
2、调用search方法从Bucket的根节点开始查找;

我们来看看search方法:

```
//boltdb/bolt/cursor.go

// search recursively performs a binary search against a given page/node until it finds a given key.
func (c *Cursor) search(key []byte, pgid pgid) {
    p, n := c.bucket.pageNode(pgid)
    if p != nil && (p.flags&(branchPageFlag|leafPageFlag)) == 0 {
        panic(fmt.Sprintf("invalid page type: %d: %x", p.id, p.flags))
    }
    e := elemRef{page: p, node: n}
    c.stack = append(c.stack, e)

    // If we're on a leaf page/node then find the specific node.
    if e.isLeaf() {
        c.nsearch(key)                                                     (1)
        return
    }

    if n != nil {
        c.searchNode(key, n)                                               (2)
        return
    }
    c.searchPage(key, p)                                                   (3)
}
```

在search中:

1、通过Bucket的pageNode()方法根据pgid找到缓存在Bucket中的node或者还未实例化的page;
2、创建一个elemRef对象，它指向pgid指定的B+Tree节点，index为0;
3、将刚创建的elemRef对象添加到stack的尾部，实现将Cursor移动到相应的B+Tree节点;
4、如果游标指向叶子节点，则代码(1)处将开始在节点内部查找;
5、如果游标指向根节点或者内节点，且pgid有对应的node对象，则代码(3)处开始在非叶子节点中查找，以确定子节点并进行递归查找;
6、如果游标指向根节点或者内节点，且pgid没有对应的node，只能从page中直接读branchPageElement，由代码(4)处开始在非叶子节点中查找，以确定子节点并进行递归查找;

Bucket的pageNode()方法的思想是: 从Bucket缓存的nodes中查找对应pgid的node，若有则返回node，page为空；若未找到，再从Bucket所属的Transaction申请过的page的集合(Tx的pages字段)中查找，有则返回该page，node为空；若Transaction中的page缓存中仍然没有，则直接从DB的内存映射中查找该页，并返回。总之，pageNode()就是根据pgid找到node或者page。

接下来我们来分析c.searchNode(key, n)的过程 [c.searchPage(key, p)与之类似]

```
//boltdb/bolt/cursor.go

func (c *Cursor) searchNode(key []byte, n *node) {
    var exact bool
    index := sort.Search(len(n.inodes), func(i int) bool {
        // TODO(benbjohnson): Optimize this range search. It's a bit hacky right now.
        // sort.Search() finds the lowest index where f() != -1 but we need the highest index.
        ret := bytes.Compare(n.inodes[i].key, key)
        if ret == 0 {
            exact = true
        }
        return ret != -1
    })
    if !exact && index > 0 {
        index--
    }
    c.stack[len(c.stack)-1].index = index

    // Recursively search to the next page.
    c.search(key, n.inodes[index].pgid)
}
```

在searchNode()中:

1、对节点中inodes进行有序查找(Go中sort.Search()使用二分法查找)，结果要么是正好找到key，即exact=true且ret=0；要么是找到比key大的最小inode(即ret=1)或者所有inodes的key均小于目标key;
2、如果查找结果是上述的第二情况，则将结果索引值减1，因为欲查找的key肯定位于前一个Pointer对应的子节点或子树中，可以对照我们前面举的BucketAA的例子进行理解;
3、将stack中的最后一个elemRef元素中的index置为查找结果索引值，即将游标移动到所指节点中具体的inode处;
4、从inode记录的pgid(可以理解为Pointer)对应的子节点处开始递归查找;

递归调用的结束条件是游标最终移动到一个叶子节点处，进入c.nsearch(key)过程:

```
//boltdb/bolt/cursor.go

// nsearch searches the leaf node on the top of the stack for a key.
func (c *Cursor) nsearch(key []byte) {
    e := &c.stack[len(c.stack)-1]
    p, n := e.page, e.node

    // If we have a node then search its inodes.
    if n != nil {
        index := sort.Search(len(n.inodes), func(i int) bool {
            return bytes.Compare(n.inodes[i].key, key) != -1
        })
        e.index = index
        return
    }

    // If we have a page then search its leaf elements.
    inodes := p.leafPageElements()
    index := sort.Search(int(p.count), func(i int) bool {
        return bytes.Compare(inodes[i].key(), key) != -1
    })
    e.index = index
}
```

在nsearch()中:

1、解出游标指向节点的node或者page;
2、如果是node，则在node内进行有序查找，结果为正好找到key或者未找到，未找到时index位于节点结尾处;
3、如果是page，则在page内进行有序查找，结果也是正好找到key或者未找到，未找到时index位于节点结尾于;
4、将当前游标元素的index置为刚刚得到的查找结果索引值，即把游标移动到key所在节点的具体inode处或者结尾处;

到此，我们就了解了Bucket通过Cursor进行查找的过程，我们回头看看seek()方法，在查找结束后，如果未找到key，即key指向叶子结点的结尾处，则它返回空；如果找到则返回K/V对。需要注意的是，在再次调用Cursor的seek()方法移动游标之前，游标一直位于上次seek()调用后的位置。

我们再回到在CreateBucket()的实现中，其中代码(3)处创建了一个内置的子Bucket，它的Bucket头bucket中的root为0，接着代码(3)处通过Bucket的write()方法将内置Bucket序列化，我们可以通过它了解内置Bucket是如何作为Value存于页框上的:

```
//boltdb/bolt/bucket.go

// write allocates and writes a bucket to a byte slice.
func (b *Bucket) write() []byte {
    // Allocate the appropriate size.
    var n = b.rootNode
    var value = make([]byte, bucketHeaderSize+n.size())

    // Write a bucket header.
    var bucket = (*bucket)(unsafe.Pointer(&value[0]))
    *bucket = *b.bucket

    // Convert byte slice to a fake page and write the root node.
    var p = (*page)(unsafe.Pointer(&value[bucketHeaderSize]))
    n.write(p)

    return value
}
```

从实现上看，内置Bucket就是将Bucket头和其根节点作为Value存于页框上，而其头中字段均为0。接着，CreateBucket()中代码(6)处将刚创建的子Bucket插入游标位置，我们来看看其中的c.node()调用:

```
//boltdb/bolt/cursor.go

// node returns the node that the cursor is currently positioned on.
func (c *Cursor) node() *node {
    _assert(len(c.stack) > 0, "accessing a node with a zero-length cursor stack")

    // If the top of the stack is a leaf node then just return it.
    if ref := &c.stack[len(c.stack)-1]; ref.node != nil && ref.isLeaf() {
        return ref.node
    }

    // Start from root and traverse down the hierarchy.
    var n = c.stack[0].node
    if n == nil {
        n = c.bucket.node(c.stack[0].page.id, nil)
    }
    for _, ref := range c.stack[:len(c.stack)-1] {
        _assert(!n.isLeaf, "expected branch node")
        n = n.childAt(int(ref.index))
    }
    _assert(n.isLeaf, "expected leaf node")
    return n
}
```

它的基本思路是: 如果当前游标指向了一个叶子节点，且节点已经实例化为node，则直接返回该node；否则，从根节点处沿Cursor的查找路径将沿途的所有节点中未实例化的节点实例化成node，并返回最后节点的node。实际上最后的节点总是叶子节点，因为B+Tree的Key全部位于叶子节点，而内节点只用来索引，所以Cursor在seek()时，总会遍历到叶子节点。也就是说，c.node()总是返回当前游标指向的叶子节点的node引用。代码(6)处通过c.node()的调用得到当前游标处的node后，随即通过node的put()方法向node中写入K/V:

```
//boltdb/bolt/node.go

// put inserts a key/value.
func (n *node) put(oldKey, newKey, value []byte, pgid pgid, flags uint32) {

    ......

    // Find insertion index.
    index := sort.Search(len(n.inodes), func(i int) bool { return bytes.Compare(n.inodes[i].key, oldKey) != -1 })

    // Add capacity and shift nodes if we don't have an exact match and need to insert.
    exact := (len(n.inodes) > 0 && index < len(n.inodes) && bytes.Equal(n.inodes[index].key, oldKey))
    if !exact {
        n.inodes = append(n.inodes, inode{})
        copy(n.inodes[index+1:], n.inodes[index:])
    }

    inode := &n.inodes[index]
    inode.flags = flags
    inode.key = newKey
    inode.value = value
    inode.pgid = pgid
    _assert(len(inode.key) > 0, "put: zero-length inode key")
}
```

在node的put()方法中:

1、先查找oldKey以确定插入的位置。要么是正好找到oldKey，则插入位置就是oldKey的位置，实际上是替换；要么node中不含有oldKey，则插入位置就是第一个大于oldKey的前一个key的位置或者是节点结尾;
2、如果找到oldKey，则直接用newKey和value替代原来的oldKey和old value；
3、如果未找到oldKey，则向node中添加一个inode，并将插入位置后的所有inode向后移一位，以实现插入新的inode;

到此，我们就了解了整个Bucket创建的主要过程：

1、根据Bucket的名字(key)搜索父Bucket，以确定表示Bucket的K/V对的插入位置;
2、创建空的内置Bucket，并将它序列化成byte slice，以作为Bucket的Value;
3、将表示Bucket的K/V对(即inode的flags为bucketLeafFlag(0x01))写入父Bucket的叶子节点;
4、通过Bucket的Bucket(name []byte)在父Bucket中查找刚刚创建的子Bucket，在了解了Bucket通过Cursor进行查找和遍历的过程后，读者可以尝试自行分析这一过程，我们在后续文中还会介绍Bucket的Get()方法，与此类似。

创建完Bucket后，就可以调用它的Put()方法向其中添加K/V了

```
//boltdb/bolt/node.go

// Put sets the value for a key in the bucket.
// If the key exist then its previous value will be overwritten.
// Supplied value must remain valid for the life of the transaction.
// Returns an error if the bucket was created from a read-only transaction, if the key is blank, if the key is too large, or if the value is too large.
func (b *Bucket) Put(key []byte, value []byte) error {
    
    ......

    // Move cursor to correct position.
    c := b.Cursor()
    k, _, flags := c.seek(key)

    // Return an error if there is an existing key with a bucket value.
    if bytes.Equal(key, k) && (flags&bucketLeafFlag) != 0 {
        return ErrIncompatibleValue
    }

    // Insert into node.
    key = cloneBytes(key)
    c.node().put(key, key, value, 0, 0)

    return nil
}
```

可以看到，向Bucket添加K/V的过程与创建Bucket时的过程极其相似，因为创建Bucket实际上就是向父Bucket添加一个标识为Bucket的K/V对而已。不同的是，这里作了一个保护，即当欲插入的Key与当前Bucket中已有的一个子Bucket的Key相同时，会拒绝写入，从而保护嵌套的子Bucket的引用不会被擦除，防止子Bucket变成孤儿。当然，我们也可以调用新创建的子Bucket的CreateBucket()方法来创建孙Bucket，然后向孙Bucket中写入K/V对，其过程与我们上述从根Bucket创建子Bucket的过程是一致的。

## 修改DB时步骤

通过分析Commit()的代码来了解修改DB时将执行哪些步骤

```
//boltdb/bolt/tx.go

// Commit writes all changes to disk and updates the meta page.
// Returns an error if a disk write error occurs, or if Commit is
// called on a read-only transaction.
func (tx *Tx) Commit() error {
    
    ......

    // TODO(benbjohnson): Use vectorized I/O to write out dirty pages.

    // Rebalance nodes which have had deletions.
    var startTime = time.Now()
    tx.root.rebalance()                                                 (1)
    ......

    // spill data onto dirty pages.
    startTime = time.Now()
    if err := tx.root.spill(); err != nil {                             (2)
        tx.rollback()
        return err
    }
    tx.stats.SpillTime += time.Since(startTime)

    // Free the old root bucket.
    tx.meta.root.root = tx.root.root                                    (3)

    opgid := tx.meta.pgid                                               (4)

    // Free the freelist and allocate new pages for it. This will overestimate
    // the size of the freelist but not underestimate the size (which would be bad).
    tx.db.freelist.free(tx.meta.txid, tx.db.page(tx.meta.freelist))
    p, err := tx.allocate((tx.db.freelist.size() / tx.db.pageSize) + 1) (5)
    if err != nil {
        tx.rollback()
        return err
    }
    if err := tx.db.freelist.write(p); err != nil {                     (6)
        tx.rollback()
        return err
    }
    tx.meta.freelist = p.id

    // If the high water mark has moved up then attempt to grow the database.
    if tx.meta.pgid > opgid {                                           (7)
        if err := tx.db.grow(int(tx.meta.pgid+1) * tx.db.pageSize); err != nil {  (8)
            tx.rollback()
            return err
        }
    }                                                                                 

    // Write dirty pages to disk.
    startTime = time.Now()
    if err := tx.write(); err != nil {                                  (9)
        tx.rollback()
        return err
    }

    ......

    // Write meta to disk.
    if err := tx.writeMeta(); err != nil {                              (10)
        tx.rollback()
        return err
    }
    tx.stats.WriteTime += time.Since(startTime)

    // Finalize the transaction.
    tx.close()                                                          (11)

    // Execute commit handlers now that the locks have been removed.
    for _, fn := range tx.commitHandlers {                                           
        fn()                                                            (12)
    }

    return nil
}
```

在Commit()中:

1、代码(1)处对根Bucket进行再平衡，这里的根Bucket也是整个DB的根Bucket，然而从根Bucket进行再平衡并不是要对DB中所有节点进行操作，而且对当前读写Transaction访问过的Bucket中的有删除操作的节点进行再平衡;
2、代码(2)处对根Bucket进行溢出操作，同样地，也是对访问过的子Bucket进行溢出操作，而且只有当节点中Key数量确实超限时才会分裂节点;
3、进行再旋转与分裂后，根Bucket的根节点可能发生了变化，因此代码(3)处将根Bucket的根节点的页号更新，且最终会写入DB的meta page;
4、代码(5)~(6)处更新DB的freeList page， 这里需要解释一下为什么对freelist作了先释放后重新分配页框并写入的操作，这是因为在代码(9)处写磁盘时，只会向磁盘写入由当前Transaction分配并写入过的页(脏页)，由于freeList page最初是在db初始化过程中分配的页，如果不在Transaction内释放并重新分配，那么freeList page将没有机会被更新到DB文件中，这里的实现并不很优雅，读者可以想一想更好的实现方式;
5、代码(4)、(7)和(8)是为了实现这样的逻辑: 只有当映射入内存的页数增加时，才调用db.grow()来刷新磁盘文件的元数据，以及时更新文件大小信息。这里需要解释一下: 我们前面介绍过， windows平台下db.mmap()调用会通过ftruncate系统调用来增加文件大小，而linux平台则没有，但linux平台会在db.grow()中调用ftruncate更新文件大小。我们前面介绍过，BoltDB写数据时不是通过mmap内存映射写文件的，而是直接通过fwrite和fdatesync系统调用 向文件系统写文件。当向文件写入数据时，文件系统上该文件结点的元数据可能不会立即刷新，导致文件的size不会立即更新，当进程crash时，可能会出现写文件结束但文件大小没有更新的情况，所以为了防止这种情况出现，在写DB文件之前，无论是windows还是linux平台，都会通过ftruncate系统调用来增加文件大小；但是linux平台为什么不在每次mmap的时候调用ftruncate来更新文件大小呢？这里是一个优化措施，因为频繁地ftruncate系统调用会影响性能，这里的优化方式是: 只有当：1) 重新分配freeListPage时，没有空闲页，这里大家可能会有疑惑，freeListPage不是刚刚通过freelist的free()方法释放过它所占用的页吗，还会有重新分配时没有空闲页的情况吗？实际上，free()过并不是真正释放页，而是将页标记为pending，要等创建下一次读写Transaction时才会被真正回收(大家可以查看freeist的free()和release()以及DB的beginRWTx()方法中最后一节代码了解具体逻辑)；2) remmap的长度大于文件实际大小时，才会调用ftruncate来增加文件大小，且当映射文件大小大于16M后，每次增加文件大小时会比实际需要的文件大小多增加16M。这里的优化比较隐晦，实现上并不优雅，读者也可以思考一下更好的优化方式;
6、代码(9)处将当前transaction分配的脏页写入磁盘;
7、代码(10)处将当前transaction的meta写入DB的meta页，因为进行读写操作后，meta中的txid已经改变，root、freelist和pgid也有可能已经更新了;
8、代码(11)处关闭当前transaction，清空相关字段;
9、代码(12)处回调commit handlers;

接下来，我们先来看看Bucket的rebalance()方法:

```
//boltdb/bolt/bucket.go

// rebalance attempts to balance all nodes.
func (b *Bucket) rebalance() {
    for _, n := range b.nodes {
        n.rebalance()
    }
    for _, child := range b.buckets {
        child.rebalance()
    }
}
```

它先对Bucket中缓存的node进行再平衡操作，然后对所有子Bucket递归调用rebalance()。node的rebalance():

```
//boltdb/bolt/node.go

// rebalance attempts to combine the node with sibling nodes if the node fill
// size is below a threshold or if there are not enough keys.
func (n *node) rebalance() {
    if !n.unbalanced {                                                            (1)
        return
    }
    n.unbalanced = false

    ......

    // Ignore if node is above threshold (25%) and has enough keys.
    var threshold = n.bucket.tx.db.pageSize / 4
    if n.size() > threshold && len(n.inodes) > n.minKeys() {                      (2)
        return
    }

    // Root node has special handling.
    if n.parent == nil {
        // If root node is a branch and only has one node then collapse it.
        if !n.isLeaf && len(n.inodes) == 1 {                                      (3)
            // Move root's child up.
            child := n.bucket.node(n.inodes[0].pgid, n)
            n.isLeaf = child.isLeaf
            n.inodes = child.inodes[:]
            n.children = child.children

            // Reparent all child nodes being moved.
            for _, inode := range n.inodes {
                if child, ok := n.bucket.nodes[inode.pgid]; ok {
                    child.parent = n
                }
            }

            // Remove old child.
            child.parent = nil
            delete(n.bucket.nodes, child.pgid)
            child.free()
        }

        return
    }

    // If node has no keys then just remove it.
    if n.numChildren() == 0 {                                                     (4)
        n.parent.del(n.key)
        n.parent.removeChild(n)
        delete(n.bucket.nodes, n.pgid)
        n.free()
        n.parent.rebalance()
        return
    }

    _assert(n.parent.numChildren() > 1, "parent must have at least 2 children")

    // Destination node is right sibling if idx == 0, otherwise left sibling.
    var target *node
    var useNextSibling = (n.parent.childIndex(n) == 0)                            (5)
    if useNextSibling {
        target = n.nextSibling()
    } else {
        target = n.prevSibling()
    }

    // If both this node and the target node are too small then merge them.
    if useNextSibling {                                                           (6)
        // Reparent all child nodes being moved.
        for _, inode := range target.inodes {
            if child, ok := n.bucket.nodes[inode.pgid]; ok {
                child.parent.removeChild(child)
                child.parent = n
                child.parent.children = append(child.parent.children, child)
            }
        }

        // Copy over inodes from target and remove target.
        n.inodes = append(n.inodes, target.inodes...)
        n.parent.del(target.key)
        n.parent.removeChild(target)
        delete(n.bucket.nodes, target.pgid)
        target.free()
    } else {                                                                      (7)
        // Reparent all child nodes being moved.
        for _, inode := range n.inodes {
            if child, ok := n.bucket.nodes[inode.pgid]; ok {
                child.parent.removeChild(child)
                child.parent = target
                child.parent.children = append(child.parent.children, child)
            }
        }

        // Copy over inodes to target and remove node.
        target.inodes = append(target.inodes, n.inodes...)
        n.parent.del(n.key)
        n.parent.removeChild(n)
        delete(n.bucket.nodes, n.pgid)
        n.free()
    }

    // Either this node or the target node was deleted from the parent so rebalance it.
    n.parent.rebalance()                                                         (8)
}
```

它的逻辑稍微有些复杂，主要包含:

1、代码(1)处限制只有unbalanced为true时才进行节点再平衡，只有当节点中有过删除操作时，unbalanced才为true;
2、代码(2)处作再平衡条件检查，只有当节点存的K/V总大小小于页大小的25%且节点中Key的数量少于设定的每节点Key数量最小值时，才会进行旋转;
3、代码(3)处处理当根节点只有一个子节点的情形(可以回顾上图中最后一步): 将子节点上的inodes拷贝到根节点上，并将子节点的所有孩子移交给根节点，并将孩子节点的父节点更新为根节点；如果子节点是一个叶子节点，则将子节点的inodes全部拷贝到根节点上后，根节点也是一个叶子节点；最后，将子节点删除;
4、如果节点变成一个空节点，则将它从B+Tree中删除，并把父节点上的Key和Pointer删除，由于父节点上有删除，得对父节点进行再平衡;
5、代码(5)~(7)合并兄弟节点与当前节点中的记录集合。代码(3)处决定是合并左节点还是右节点：如果当前节点是父节点的第一个孩子，则将右节点中的记录合并到当前节点中；如果当前节点是父节点的第二个或以上的节点，则将当前节点中的记录合并到左节点中;
6、代码(6)将右节点中记录合并到当前节点：首先将右节点的孩子节点全部变成当前节点的孩子，右节点将所有孩子移除；随后，将右节点中的记录全部拷贝到当前节点；最后，将右节点从B+Tree中移除，并将父节点中与右节点对应的记录删除;
7、代码(7)将当前节点中的记录合并到左节点，过程与代码(6)处类似;
合并兄弟节点与当前节点时，会移除一个节点并从父节点中删除一个记录，所以需要对父节点进行8、再平衡，如代码(8)处所示，所以节点的rebalance也是一个递归的过程，它会从当前结点一直进行到根节点处;

在Bucket rebalance过程中，节点合并后其大小可能超过页大小，但在接下来的spill过程中，超过页大小的节点会进行分裂。接下来，我们来看看Bucket的spill()方法:

```
//boltdb/bolt/bucket.go

// spill writes all the nodes for this bucket to dirty pages.
func (b *Bucket) spill() error {
    // Spill all child buckets first.
    for name, child := range b.buckets {
        // If the child bucket is small enough and it has no child buckets then
        // write it inline into the parent bucket's page. Otherwise spill it
        // like a normal bucket and make the parent value a pointer to the page.
        var value []byte
        if child.inlineable() {                                         (1)
            child.free()
            value = child.write()
        } else {
            if err := child.spill(); err != nil {                       (2)
                return err
            }

            // Update the child bucket header in this bucket.
            value = make([]byte, unsafe.Sizeof(bucket{}))               (3)
            var bucket = (*bucket)(unsafe.Pointer(&value[0]))           (4)
            *bucket = *child.bucket                                     (5)
        }

        // Skip writing the bucket if there are no materialized nodes.
        if child.rootNode == nil {
            continue
        }

        // Update parent node.
        var c = b.Cursor()
        k, _, flags := c.seek([]byte(name))
        
        ......
        
        c.node().put([]byte(name), []byte(name), value, 0, bucketLeafFlag)      (6)
    }

    // Ignore if there's not a materialized root node.
    if b.rootNode == nil {
        return nil
    }

    // Spill nodes.
    if err := b.rootNode.spill(); err != nil {                          (7)
        return err
    }
    b.rootNode = b.rootNode.root()                                      (8)

    // Update the root node for this bucket.
    if b.rootNode.pgid >= b.tx.meta.pgid {
        panic(fmt.Sprintf("pgid (%d) above high water mark (%d)", b.rootNode.pgid, b.tx.meta.pgid))
    }
    b.root = b.rootNode.pgid                                            (9)

    return nil
}
```

在spill()中:

1、首先，对Bucket树的子Bucket进行深度优先访问并递归调用spill();
2、在代码(2)处，子Bucket不满足inlineable()条件时，如果子Bucket原来是一个内置Bucket，则它将通过spill()变成一个普通的Bucket，即它的B+Tree有一个根节点和至少两个叶子节点；如果子Bucket原本是一个普通Bucket，则spill()可能会更新它的根节点。从代码(3)~(5)可以看出，一个普通的子Bucket的Value只保存了Bucket的头部。相反地，在代码(1)处，如果一个普通的子Bucket由于K/V记录减少而满足了inlineable()条件时，它将变成一个内置Bucket，即它的B+Tree只有一个根节点，并将根节点上的所有inodes作为Value写入父Bucket;
3、代码(6)处将子Bucket的新的Value更新到父Bucket中;
4、更新完子Bucket后，就开始spill自己，代码(7)处从当前Bucket的根节点处开始spill。在递归的最内层调用中，访问到了Bucket树的某个(逻辑)叶子Bucket，由于它没有子Bucket，将直接从其根节开始spill;
5、Bucket spill完后，其根节点可能有变化，所以代码(8)处更新根节点引用;
6、最后，代码(9)处更新Bucket头中的根节点页号;

我们先来通过inlineable()的代码了解成为内置Bucket的条件:

```
//boltdb/bolt/bucket.go

// inlineable returns true if a bucket is small enough to be written inline
// and if it contains no subbuckets. Otherwise returns false.
func (b *Bucket) inlineable() bool {
    var n = b.rootNode

    // Bucket must only contain a single leaf node.
    if n == nil || !n.isLeaf {
        return false
    }

    // Bucket is not inlineable if it contains subbuckets or if it goes beyond
    // our threshold for inline bucket size.
    var size = pageHeaderSize
    for _, inode := range n.inodes {
        size += leafPageElementSize + len(inode.key) + len(inode.value)

        if inode.flags&bucketLeafFlag != 0 {
            return false
        } else if size > b.maxInlineBucketSize() {
            return false
        }
    }

    return true
}

......

// Returns the maximum total size of a bucket to make it a candidate for inlining.
func (b *Bucket) maxInlineBucketSize() int {
    return b.tx.db.pageSize / 4
}
```

可以看出，只有当Bucket只有一个叶子节点(即其根节点)且它序列化后的大小小于页大小的25%时才能成为内置Bucket。接下来，我们开始分析node的spill过程:

```
//boltdb/bolt/node.go

// spill writes the nodes to dirty pages and splits nodes as it goes.
// Returns an error if dirty pages cannot be allocated.
func (n *node) spill() error {
    var tx = n.bucket.tx
    if n.spilled {                                                      (1)
        return nil
    }

    // Spill child nodes first. Child nodes can materialize sibling nodes in
    // the case of split-merge so we cannot use a range loop. We have to check
    // the children size on every loop iteration.
    sort.Sort(n.children)
    for i := 0; i < len(n.children); i++ {                              (2)
        if err := n.children[i].spill(); err != nil {
            return err
        }
    }

    // We no longer need the child list because it's only used for spill tracking.
    n.children = nil                                                    (3)

    // Split nodes into appropriate sizes. The first node will always be n.
    var nodes = n.split(tx.db.pageSize)                                 (4)
    for _, node := range nodes {
        // Add node's page to the freelist if it's not new.
        if node.pgid > 0 {
            tx.db.freelist.free(tx.meta.txid, tx.page(node.pgid))       (5)
            node.pgid = 0
        }

        // Allocate contiguous space for the node.
        p, err := tx.allocate((node.size() / tx.db.pageSize) + 1)       (6)
        if err != nil {
            return err
        }

        // Write the node.
        if p.id >= tx.meta.pgid {
            panic(fmt.Sprintf("pgid (%d) above high water mark (%d)", p.id, tx.meta.pgid))
        }
        node.pgid = p.id                                                (7)
        node.write(p)                                                   (8)
        node.spilled = true                                             (9)

        // Insert into parent inodes.
        if node.parent != nil {
            var key = node.key
            if key == nil {
                key = node.inodes[0].key                                (10)     
            }

            node.parent.put(key, node.inodes[0].key, nil, node.pgid, 0) (11)
            node.key = node.inodes[0].key                               (12)
            _assert(len(node.key) > 0, "spill: zero-length node key")
        }

        // Update the statistics.
        tx.stats.Spill++
    }

    // If the root node split and created a new root then we need to spill that
    // as well. We'll clear out the children to make sure it doesn't try to respill.
    if n.parent != nil && n.parent.pgid == 0 {
        n.children = nil                                                (13)
        return n.parent.spill()                                         (14)
    }

    return nil
}
```

node的spill过程与rebalance过程不同，rebalance是从当前节点到根节点递归，而spill是从根节点到叶子节点进行递归，不过它们最终都要处理根节点的rebalance或者spill。在spill中，如果根节点需要分裂(如上图中的第二步)，则需要对其递归调用spill，但是为了防止循环，调用父节点的spill之前，会将父node中缓存的子节点引用集合children置空，以防止向下递归。我们来看看它的具体实现：

1、代码(1)处检测当前node是否已经spill过，如果spill过了则无需spill;
2、代码(2)处对子节点进行深度优先访问并递归调用spill()，需要注意的是，子节点可能会分裂成多个节点，分裂出来的新节点也是当前节点的子节点，n.children这个slice的size会在循环中变化，帮不能使用rang的方式循环访问；同时，分裂出来的新节点会在代码(9)处被设为spilled，所以在代码(2)的下一次循环访问到新的子节点时不会重新spill，这也是代码(1)处对spilled进行检查的原因;
3、当所有子节点spill完成后，代码(3)处将子节点引用集合children置为空，以防向上递归调用spill的时候形成回路;
4、代码(4)处调用node的split()方法按页大小将node分裂出若干新node，新node与当前node共享同一个父node，返回的nodes中包含当前node;
5、随后代码(5)~(12)处理分裂后产生的node。代码(5)处为释放当前node的所占页，因为随后要为它分配新的页，我们前面说过transaction commit是只会向磁盘写入当前transaction分配的脏页，所以这里要对当前node重新分配页;
6、代码(6)处调用Tx的allocate()方法为分裂产生的node分配页缓存，请注意，通过splite()方法分裂node后，node的大小为页大小 * 填充率， 默认填充率为50%， 而且一般地它的值小于100%， 所以这里为每个node实际上是分配一个页框;
7、代码(7)处将新node的页号设为分配给他的页框的页号，同时，代码(8)处将新node序列化并写入刚刚分配的页缓存;
8、代码(9)处将spilled设为true，我们刚刚介绍过它的作用;
9、代码(10)~(12)处向父节点更新或添加Key和Pointer，以指向分裂产生的新node。代码(10)将父node的key设为第一个子node的第一个key;
10、代码(11)处向父node写入Key和Pointer，其中Key是子结点的第一个key，Pointer是子节点的pgid;
11、代码(12)处将分裂产生的node的key设为其中的第一个key;
12、从根节点处递归完所有子节点的spill过程后，若根节点需要分裂，则它分裂后将产生新的根节点，代码(13)和(14)对新产生的根节点进行spill;

在node的spill()过程中，除了通过递归来保证整个树结构被spill外，比较重要的地方是spite如何分裂节点，我们来看看node的split()方法:

```
//boltdb/bolt/node.go

// split breaks up a node into multiple smaller nodes, if appropriate.
// This should only be called from the spill() function.
func (n *node) split(pageSize int) []*node {
    var nodes []*node

    node := n
    for {
        // Split node into two.
        a, b := node.splitTwo(pageSize)
        nodes = append(nodes, a)

        // If we can't split then exit the loop.
        if b == nil {
            break
        }

        // Set node to b so it gets split on the next iteration.
        node = b
    }

    return nodes
}
```

可以看到，split()实际上就是把node分成两段，其中一段满足node要求的大小，另一段再进一步按相同规则分成两段，一直到不能再分为止。我们来看看其中的splitTwo():

```
//boltdb/bolt/node.go

// splitTwo breaks up a node into two smaller nodes, if appropriate.
// This should only be called from the split() function.
func (n *node) splitTwo(pageSize int) (*node, *node) {
    // Ignore the split if the page doesn't have at least enough nodes for
    // two pages or if the nodes can fit in a single page.
    if len(n.inodes) <= (minKeysPerPage*2) || n.sizeLessThan(pageSize) {   (1)
        return n, nil
    }

    // Determine the threshold before starting a new node.
    var fillPercent = n.bucket.FillPercent
    if fillPercent < minFillPercent {
        fillPercent = minFillPercent
    } else if fillPercent > maxFillPercent {
        fillPercent = maxFillPercent
    }
    threshold := int(float64(pageSize) * fillPercent)                     (2)

    // Determine split position and sizes of the two pages.
    splitIndex, _ := n.splitIndex(threshold)                              (3)

    // Split node into two separate nodes.
    // If there's no parent then we'll need to create one.
    if n.parent == nil {
        n.parent = &node{bucket: n.bucket, children: []*node{n}}          (4)
    }

    // Create a new node and add it to the parent.
    next := &node{bucket: n.bucket, isLeaf: n.isLeaf, parent: n.parent}   (5)
    n.parent.children = append(n.parent.children, next)

    // Split inodes across two nodes.
    next.inodes = n.inodes[splitIndex:]                                   (6)
    n.inodes = n.inodes[:splitIndex]                                      (7)

    ......

    return n, next
}
```

可以看到:

1、代码(1)处决定了节点分裂的条件: 1) 节点大小超过了页大小， 且2) 节点Key个数大于每节点Key数量最小值的两倍，这是为了保证分裂出的两个节点中的Key数量都大于每节点Key数量的最小值;
2、代码(2)处决定分裂的门限值，即页大小 x 填充率;
3、代码(3)处调用splitIndex()方法根据门限值计算分裂的位置;
4、代码(4)如果要分裂的节点没有父节点(可能是根节点)，则应该新建一个父node，同时将当前节点设为它的子node;
5、代码(5)创建了一个新node，并将当前node的父节点设为它的父节点;
6、代码(6)处将当前node的从分裂位置开始的右半部分记录拷贝给新node;
7、代码(7)处将当前node的记录更新为原记录集合从分裂位置开始的左半部分，从而实现了将当前node一分为二;

我们来看看splitIndex()是如何决定分裂位置的:

```
//boltdb/bolt/node.go

// splitIndex finds the position where a page will fill a given threshold.
// It returns the index as well as the size of the first page.
// This is only be called from split().
func (n *node) splitIndex(threshold int) (index, sz int) {
    sz = pageHeaderSize

    // Loop until we only have the minimum number of keys required for the second page.
    for i := 0; i < len(n.inodes)-minKeysPerPage; i++ {
        index = i
        inode := n.inodes[i]
        elsize := n.pageElementSize() + len(inode.key) + len(inode.value)

        // If we have at least the minimum number of keys and adding another
        // node would put us over the threshold then exit and return.
        if i >= minKeysPerPage && sz+elsize > threshold {
            break
        }

        // Add the element size to the total size.
        sz += elsize
    }

    return
}
```

可以看出分裂位置要同时保证:

1、前半部分的节点数量大于每节点Key数量最小值(minKeysPerPage);
2、后半部分的节点数量大于每节点Key数量最小值(minKeysPerPage);
3、分裂后半部分node的大小是不超过门限值的最小值，即前半部分的size要在门限范围内尽量大;

对Bucket进行rebalance和spill后，Bucket及其子Bucket对应的B+Tree将处于平衡状态，随后各node将被写入DB文件。这两个过程在树结构上进行递归，可能不太好理解，读者可以对照本文开头给出的示例图推演。node的spill()调用中，还涉及到了Tx的allocate()方法，我们将在介绍完Tx的Commit()后再来分析它。Commit()中接下来比较重要的步骤例是调用tx.write()和tx.writeMeta()来写DB文件了。我们先来看看tx.write():

```
//boltdb/bolt/tx.go

// write writes any dirty pages to disk.
func (tx *Tx) write() error {
    // Sort pages by id.
    pages := make(pages, 0, len(tx.pages))
    for _, p := range tx.pages {
        pages = append(pages, p)                                             (1)
    }
    // Clear out page cache early.
    tx.pages = make(map[pgid]*page)                                          (2)
    sort.Sort(pages)                                                         (3)

    // Write pages to disk in order.
    for _, p := range pages {                                                (4)
        size := (int(p.overflow) + 1) * tx.db.pageSize
        offset := int64(p.id) * int64(tx.db.pageSize)

        // Write out page in "max allocation" sized chunks.
        ptr := (*[maxAllocSize]byte)(unsafe.Pointer(p))
        for {
            // Limit our write to our max allocation size.
            sz := size
            if sz > maxAllocSize-1 {
                sz = maxAllocSize - 1
            }

            // Write chunk to disk.
            buf := ptr[:sz]
            if _, err := tx.db.ops.writeAt(buf, offset); err != nil {        (5)
                return err
            }

            ......

            // Exit inner for loop if we've written all the chunks.
            size -= sz
            if size == 0 {
                break
            }

            // Otherwise move offset forward and move pointer to next chunk.
            offset += int64(sz)
            ptr = (*[maxAllocSize]byte)(unsafe.Pointer(&ptr[sz]))
        }
    }

    // Ignore file sync if flag is set on DB.
    if !tx.db.NoSync || IgnoreNoSync {
        if err := fdatasync(tx.db); err != nil {                          (6)
            return err
        }
    }

    ......

    return nil
}
```

tx.write()的主要步骤是:

1、首先将当前tx中的脏页的引用保存到本地slice变量中，并释放原来的引用。请注意，Tx对象并不是线程安全的，而接下来的写文件操作会比较耗时，此时应该避免tx.pages被修改;
2、代码(3)处对页按其pgid排序，保证在随后按页顺序写文件，一定程度上提高写文件效率;
3、代码(4)处开始将各页循环写入文件，循环体中代码(5)处通过fwrite系统调用写文件;
4、代码(6)处通过fdatasync将磁盘缓冲写入磁盘。

在将脏页写入磁盘后，tx.Commit()随后将meta写入磁盘:

```
//boltdb/bolt/tx.go

// writeMeta writes the meta to the disk.
func (tx *Tx) writeMeta() error {
    // Create a temporary buffer for the meta page.
    buf := make([]byte, tx.db.pageSize)
    p := tx.db.pageInBuffer(buf, 0)
    tx.meta.write(p)

    // Write the meta page to file.
    if _, err := tx.db.ops.writeAt(buf, int64(p.id)*int64(tx.db.pageSize)); err != nil {
        return err
    }
    if !tx.db.NoSync || IgnoreNoSync {
        if err := fdatasync(tx.db); err != nil {
            return err
        }
    }

    ......

    return nil
}
```

writeMeta()的实现比较简单，先向临时分配的页缓存写入序列化后的meta页，然后通过fwrite和fdatesync系统调用将其写入DB的meta page。

到此，我们就完整地了解了Transaction Commit的全部过程，主要包括:

1、从根Bucket开始，对访问过的Bucket进行转换与分裂，让进行过插入与删除操作的B+Tree重新达到平衡状态;
2、更新freeList页;
3、将由当前transaction分配的页缓存写入磁盘，需要分配页缓存的地方有:          1)节点分裂时产生新的节点, 2) freeList页重新分配;
4、将meta页写入磁盘;

我们来看一看tx是如何通过allocate()方法来分配页缓存的:

```
//boltdb/bolt/tx.go

// allocate returns a contiguous block of memory starting at a given page.
func (tx *Tx) allocate(count int) (*page, error) {
    p, err := tx.db.allocate(count)
    if err != nil {
        return nil, err
    }

    // Save to our page cache.
    tx.pages[p.id] = p

    ......

    return p, nil
}
```

可以看出，它实际上是通过DB的allocate()方法来做实际分配的；同时，这里分配的页缓存将被加入到tx的pages字段中，也就是我们前面提到的脏页集合。

```
//boltdb/bolt/db.go

// allocate returns a contiguous block of memory starting at a given page.
func (db *DB) allocate(count int) (*page, error) {
    // Allocate a temporary buffer for the page.
    var buf []byte
    if count == 1 {
        buf = db.pagePool.Get().([]byte)                                 (1)
    } else {
        buf = make([]byte, count*db.pageSize)
    }
    p := (*page)(unsafe.Pointer(&buf[0]))
    p.overflow = uint32(count - 1)

    // Use pages from the freelist if they are available.
    if p.id = db.freelist.allocate(count); p.id != 0 {                   (2)
        return p, nil
    }

    // Resize mmap() if we're at the end.
    p.id = db.rwtx.meta.pgid                                             (3)
    var minsz = int((p.id+pgid(count))+1) * db.pageSize                  (4)
    if minsz >= db.datasz {                                  
        if err := db.mmap(minsz); err != nil {                           (5)
            return nil, fmt.Errorf("mmap allocate error: %s", err)
        }
    }

    // Move the page id high water mark.
    db.rwtx.meta.pgid += pgid(count)                                     (6)

    return p, nil
}
```

可以看出，实际分配页缓存的过程是:

1、首先分配所需的缓存。这里有一个优化措施: 如果只需要一页缓存的话，并不直接进行内存分配，而是通过Go中的Pool缓冲池来分配，以减小分配内存带来的时间开销。tx.write()中向磁盘写入脏页后，会将所有只占一个页框的脏页清空，并放入Pool缓冲池;
2、从freeList查看有没有可用的页号，如果有则分配给刚刚申请到的页缓存，并返回；如果freeList中没有可用的页号，则说明当前映射入内存的文件段没有空闲页，需要增大文件映射范围;
3、代码(3)处将新申请的页缓存的页号设为文件内容结尾处的页号; 请注意，我们所说的文件内容结尾处并不是指文件结尾处，如大小为32K 的文件，只写入了4页(页大小为4K)，则文件内容结尾处为16K处，结尾处的页号是4。我们前面介绍过meta.pgid简单理解为文件总页数，实际上并不准确，我们说简单理解为文件总页数，是假设文件被写满(如刚创建DB文件时)的情况。现在我们知道，当映射文件大小大于16M时，文件实际大小会大于文件内容长度。实际上，BoltDB允许在Open()的时候指定初始的文件映射长度，并可以超过文件大小，在linux平台上，在读写transaction commit之前，映射区长度均会大于文件实际大小，但meta.pgid总是记录文件内容所占的最大页号加1;
4、代码(4)处计算需要的总页数;
5、在代码(5)处，如果需要的页数大于已经映射到内存的文件总页数，则触发remmap，将映射区域扩大到写入文件后新的文件内容结尾处。我们前面介绍db.mmaplock的时候说过，读写transaction在remmap时，需要等待所有已经open的只读transaction结束，从这里我们知道，如果打开DB文件时，设定的初始文件映射长度足够长，可以减少读写transaction需要remmap的概率，从而降低读写transaction被阻塞的概率，提高读写并发;
6、代码(6)让meta.pgid指向新的文件内容结尾处;
