# box

---

## 1. 项目介绍

box是基于[bitcask](https://riak.com/assets/bitcask-intro.pdf)论文提出的LST-Tree（日志结构合并树）设计的一个kv存储引擎

## 2. 设计

### 存储结构

每个bitcask实例就是一个目录，该目录下的文件分为`active data file`和`older data file`，某一时刻bitcask实例中只能有一个`active data file`，当该文件达到一个大小阈值时，它将被关闭，并创建一个新的活动文件。一旦一个文件被关闭(不管是有意关闭还是由于服务器退出)，它就被认为是不可变的，并且再也不会打开来写入。

![](https://tva1.sinaimg.cn/large/008i3skNgy1gxkf3rchqzj319e0u0acp.jpg)

### 内存索引

bitcask实例需要在内存中维护一个全局索引结构，`key`会存储在内存中以便快速查找，所有的`value`都存储于磁盘中，这种方法特点是以追加的方式写磁盘，即写操作是有序的，这样可以减少磁磁盘的寻道时间，是一种高吞吐量的写入方案，在更新数据时，也是把新数据追加到文件的后面，然后更新一下数据的文件指针映射即可。

当写操作发生时，`keydir`被原子的更新，更新成新的文件上的位置，文件上的老数据还在磁盘上，新的读操作会使用新的`keydir`。

![](https://tva1.sinaimg.cn/large/008i3skNgy1gxkf0pw880j31ei0u0n0k.jpg)

#### 数据记录格式

box存储引擎通过顺序写的方式来避免随机IO提高写入效率，所以`日志`的结构的设计尤为重要。

> The active file is only written by appending, which means that sequential writes do not require disk seeking. 

* crc :循环校验码，占用4个字节

* tstamp：时间戳，占用8个字节

* ksz：key占用的字节大小，占用4个字节

* value_size：value占用的字节大小，占用4个字节

* key：key的值，占用字节数未知

* value：value的值，占用的字节数未知 

```go
| CRC 4 | TS 8  | KS 4 | VS 4  | KEY ? | VALUE ? |
```

### Merge

`Bitcask`作为一个只追加记录的引擎，怎么解决随着记录不断增多，数据文件也会变得更大的问题？为了节省空间，`bitcask`采用`merge`的方式剔除脏数据

```go
// 合并数据文件，清除无效的记录
func (db *box) Merge() error {

	if db.dbFile.Offset == 0 {
		return nil
	}

	var (
		validEntries []*Entry
		offset       int64
	)

	for {
		e, err := db.dbFile.Read(offset)
		if err != nil {
			if err == io.EOF {
				break
			}
			return err
		}
		// To find latest entry
		if off, ok := db.indexes[string(e.Key)]; ok && off == offset {
			validEntries = append(validEntries, e)
		}
		offset += e.GetSize()
	}

	if len(validEntries) > 0 {
		// 新建临时文件
		mergeDBFile, err := NewMergeDBFile(db.dirPath)
		if err != nil {
			return err
		}
		defer os.Remove(mergeDBFile.File.Name())

		// 重新写入有效的 entry
		for _, entry := range validEntries {
			writeOff := mergeDBFile.Offset
			err := mergeDBFile.Write(entry)
			if err != nil {
				return err
			}

			// 更新索引
			db.indexes[string(entry.Key)] = writeOff
		}

		// 获取文件名
		dbFileName := db.dbFile.File.Name()
		// 关闭文件
		db.dbFile.File.Close()
		// 删除旧的数据文件
		os.Remove(dbFileName)

		// 获取文件名
		mergeDBFileName := mergeDBFile.File.Name()
		// 关闭文件
		mergeDBFile.File.Close()
		// 临时文件变更为新的数据文件
		os.Rename(mergeDBFileName, db.dirPath+string(os.PathSeparator)+FileName)

		db.dbFile = mergeDBFile
	}
	return nil
}
```



## 参考

https://riak.com/assets/bitcask-intro.pdf 
