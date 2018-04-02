# ETHDB

go-ethereum所有的数据存储在[levelDB](https://github.com/google/leveldb)这个Google开源的KeyValue文件数据库中，整个区块链的所有数据都存储在一个levelDB的数据库中，levelDB支持按照文件大小切分文件的功能，所以我们看到的区块链的数据都是一个一个小文件，其实这些小文件都是一个同一个levelDB实例。

## [levelDB](https://github.com/google/leveldb)

### 特性
* Keys and values are arbitrary byte arrays
* Data is stored sorted by key
* Callers can provide a custom comparison function to override the sort order
* The basic operations are Put(key,value), Get(key), Delete(key)
* Multiple changes can be made in one atomic batch
* Users can create a transient snapshot to get a consistent view of data
* Data is automatically compressed using the Snappy compression library
* External activity (file system operations etc.) is relayed through a virtual interface so users can customize the operating system interactions

### 限制
* This is not a SQL database. It does not have a relational data model, it does not support SQL queries, and it has no support for indexes
* Only a single process (possibly multi-threaded) can access a particular database at a time.
* There is no client-server support builtin to the library. An application that needs such support will have to wrap their own server around the library.

## Ethdb code
ethdb的代码都在ethdb包下面。

* database.go levelDB的封装代码
* memory_database.go	供测试用的基于内存的数据库，不会持久化为文件，仅供测试
* interface.go	定义了数据库的接口

``` go
// Code using batches should try to add this much data to the batch.
// The value was determined empirically.
const IdealBatchSize = 100 * 1024

// Putter wraps the database write operation supported by both batches and regular databases.
// Putter接口定义了批量操作和普通操作的写入接口
type Putter interface {
	Put(key []byte, value []byte) error
}

// Database wraps all database operations. All methods are safe for concurrent use.
// 数据库接口定义了所有的数据库操作， 所有的方法都是多线程安全的
type Database interface {
	Putter
	Get(key []byte) ([]byte, error)
	Has(key []byte) (bool, error)
	Delete(key []byte) error
	Close()
	NewBatch() Batch
}

// Batch is a write-only database that commits changes to its host database
// when Write is called. Batch cannot be used concurrently.
// 批量操作接口，不能多线程同时使用，当Write方法被调用的时候，数据库会提交写入的更改
type Batch interface {
	Putter
	ValueSize() int // amount of data in the batch
	Write() error
	// Reset resets the batch for reuse
	Reset()
}
```
