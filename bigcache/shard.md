bigcache是按照不同的shard进行分段存储，这里重点看下其中的一个shard，他的定义如下：

```go
type cacheShard struct {
	hashmap     map[uint64]uint32
	entries     queue.BytesQueue
	lock        sync.RWMutex
	entryBuffer []byte
	onRemove    onRemoveCallback

	isVerbose    bool
	statsEnabled bool
	logger       Logger
	clock        clock
	lifeWindow   uint64

	hashmapStats map[uint64]uint32
	stats        Stats
}
```



#### 1.get

看下如何把数据放入shard中，首先取出它在bytes队列中的数据，然后解码出key，并判断是否等于传入的key，这里不做冲突处理，如果不相等就直接返回miss，否则取出对应的value返回。

```go
func (s *cacheShard) get(key string, hashedKey uint64) ([]byte, error) {
	s.lock.RLock()
	wrappedEntry, err := s.getWrappedEntry(hashedKey)
	if err != nil {
		s.lock.RUnlock()
		return nil, err
	}
	if entryKey := readKeyFromEntry(wrappedEntry); key != entryKey {
		s.lock.RUnlock()
		s.collision()
		if s.isVerbose {
			s.logger.Printf("Collision detected. Both %q and %q have the same hash %x", key, entryKey, hashedKey)
		}
		return nil, ErrEntryNotFound
	}
	entry := readEntry(wrappedEntry)
	s.lock.RUnlock()
	s.hit(hashedKey)

	return entry, nil
}
```



这里在看看**getWrappedEntry**首先根据hashkey获取entry，shard中对hashKey和itemIndex进行了映射，这里的entries是一个bytes队列，根据索引取出bytes。

```go
func (s *cacheShard) getWrappedEntry(hashedKey uint64) ([]byte, error) {
   itemIndex := s.hashmap[hashedKey]

   if itemIndex == 0 {
      s.miss()
      return nil, ErrEntryNotFound
   }

   wrappedEntry, err := s.entries.Get(int(itemIndex))
   if err != nil {
      s.miss()
      return nil, err
   }

   return wrappedEntry, err
}
```



接下来看看函数中的统计函数,这些函数都是用atomic原子加减

**s.miss()** 

```go
func (s *cacheShard) miss() {
	atomic.AddInt64(&s.stats.Misses, 1)
}
```

**collision()**

```go
func (s *cacheShard) collision() {
	atomic.AddInt64(&s.stats.Collisions, 1)
}
```

**hit()**

```go
func (s *cacheShard) hit(key uint64) {
	atomic.AddInt64(&s.stats.Hits, 1)
	if s.statsEnabled {
		s.lock.Lock()
		s.hashmapStats[key]++
		s.lock.Unlock()
	}
}
```



#### 2.set

接下来看下set函数，该函数先看下有没已经存储的key，如果有的话直接更新，resetKeyFromEntry函数。接下来取出队列的第一个元素判断是否过期（所以只有set的时候才有过期判断？），最后包装生成bytes队列的存储单元（wrapEntry，该函数在encoding中有详细介绍），然后存入队列。这里如果存入失败，可能是因为内存满了，然后会清理掉最旧的数据，然后把hashKey和index做一个关联。

```go
func (s *cacheShard) set(key string, hashedKey uint64, entry []byte) error {
	currentTimestamp := uint64(s.clock.epoch())

	s.lock.Lock()

	if previousIndex := s.hashmap[hashedKey]; previousIndex != 0 {
		if previousEntry, err := s.entries.Get(int(previousIndex)); err == nil {
			resetKeyFromEntry(previousEntry)
		}
	}

	if oldestEntry, err := s.entries.Peek(); err == nil {
		s.onEvict(oldestEntry, currentTimestamp, s.removeOldestEntry)
	}

	w := wrapEntry(currentTimestamp, hashedKey, key, entry, &s.entryBuffer)

	for {
		if index, err := s.entries.Push(w); err == nil {
			s.hashmap[hashedKey] = uint32(index)
			s.lock.Unlock()
			return nil
		}
		if s.removeOldestEntry(NoSpace) != nil {
			s.lock.Unlock()
			return fmt.Errorf("entry is bigger than max shard size")
		}
	}
}
```

这里看下相关的其他几个函数：

**resetKeyFromEntry** ： 该函数会把bytes中的key清空

```go
func resetKeyFromEntry(data []byte) {
	binary.LittleEndian.PutUint64(data[timestampSizeInBytes:], 0)
}
```



**onEvict**:   过期处理函数，取出存储的时间和当前时间比对，如果超过设定的时间长，就调用传入的remove函数清理。

  ```go
func (s *cacheShard) onEvict(oldestEntry []byte, currentTimestamp uint64, evict func(reason RemoveReason) error) bool {
	oldestTimestamp := readTimestampFromEntry(oldestEntry)
	if currentTimestamp-oldestTimestamp > s.lifeWindow {
		evict(Expired)
		return true
	}
	return false
}
  ```

再看看onEvict函数调用的：**removeOldestEntry** , 该函数会取出最老的entry，然后返回对应的hashIndex，并从shard的hashmap中删除，最后调用真正的onRemove函数（该函数由配置文件配置），并且把状态计数器删除。注意：这里并没有对bytes队列进行删除，所以可以理解是惰性删除。

  ```go
func (s *cacheShard) removeOldestEntry(reason RemoveReason) error {
	oldest, err := s.entries.Pop()
	if err == nil {
		hash := readHashFromEntry(oldest)
		if hash == 0 {
			// entry has been explicitly deleted with resetKeyFromEntry, ignore
			return nil
		}
		delete(s.hashmap, hash)
		s.onRemove(oldest, reason)
		if s.statsEnabled {
			delete(s.hashmapStats, hash)
		}
		return nil
	}
	return err
}
  ```



#### 3.append

append会先取出hashedKey对应的entry，然后直接在bytes里面追加entry，最后更新回去。这里可以知道，set和get的时候并没有考虑到entry有多个的情况。估计需要用户自己去处理吧。

 ```go
func (s *cacheShard) append(key string, hashedKey uint64, entry []byte) error {
	s.lock.Lock()
	var newEntry []byte
	oldEntry, err := s.getWithoutLock(key, hashedKey)
	if err != nil {
		if err != ErrEntryNotFound {
			s.lock.Unlock()
			return err
		}
	} else {
		newEntry = oldEntry
	}

	newEntry = append(newEntry, entry...)
	err = s.setWithoutLock(key, hashedKey, newEntry)
	s.lock.Unlock()
	return err
}
 ```

