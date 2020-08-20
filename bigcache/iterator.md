EntryInfoIterator的定义如下

```go
type EntryInfoIterator struct {
	mutex            sync.Mutex
	cache            *BigCache
	currentShard     int
	currentIndex     int
	currentEntryInfo EntryInfo
	elements         []uint64
	elementsCount    int
	valid            bool
}
```

其中cache是引用了要被迭代的BigCache对象

#### 1. 初始化操作

```go
func newIterator(cache *BigCache) *EntryInfoIterator {
	elements, count := cache.shards[0].copyHashedKeys()

	return &EntryInfoIterator{
		cache:         cache,
		currentShard:  0,
		currentIndex:  -1,
		elements:      elements,
		elementsCount: count,
	}
}
```

这个初始化会先找到第一个shards的所有的key, elements是一个map，key是计数，value是对应的key，并把elements的数量赋予elementsCount变量用于判断是否要进入下一个shard取值.



#### 2. 迭代函数

```go
func (it *EntryInfoIterator) SetNext() bool {
	it.mutex.Lock()

	it.valid = false
	it.currentIndex++

	if it.elementsCount > it.currentIndex {
		it.valid = true

		empty := it.setCurrentEntry()
		it.mutex.Unlock()

		if empty {
			return it.SetNext()
		} else {
			return true
		}
	}

	for i := it.currentShard + 1; i < it.cache.config.Shards; i++ {
		it.elements, it.elementsCount = it.cache.shards[i].copyHashedKeys()

		// Non empty shard - stick with it
		if it.elementsCount > 0 {
			it.currentIndex = 0
			it.currentShard = i
			it.valid = true

			empty := it.setCurrentEntry()
			it.mutex.Unlock()

			if empty {
				return it.SetNext()
			} else {
				return true
			}
		}
	}
	it.mutex.Unlock()
	return false
}
```

该函数会先判断是否要进入下一个shard获取数据，判断的依据就是elementsCount > currentIndex, 否则会去取出下一个shards的值。

最后会进入setCurrentEntry函数

```go
func (it *EntryInfoIterator) setCurrentEntry() bool {
	var entryNotFound = false
	entry, err := it.cache.shards[it.currentShard].getEntry(it.elements[it.currentIndex])

	if err == ErrEntryNotFound {
		it.currentEntryInfo = emptyEntryInfo
		entryNotFound = true
	} else if err != nil {
		it.currentEntryInfo = EntryInfo{
			err: err,
		}
	} else {
		it.currentEntryInfo = EntryInfo{
			timestamp: readTimestampFromEntry(entry),
			hash:      readHashFromEntry(entry),
			key:       readKeyFromEntry(entry),
			value:     readEntry(entry),
			err:       err,
		}
	}

	return entryNotFound
}
```

该函数会先去bigcache取出对应的entry，然后封装成EntryInfo返回（因为bigcache里面存的是一个bytes，需要反序列化出来）

如果没找到就会一直递归的去寻找。



这个比较简单，就是封装了一个EntryInfo用于Iterator的结果，然后按照shard来寻找元素