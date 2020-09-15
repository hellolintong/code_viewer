reflectx是对reflect的一个封装，他包含如下几个结构体：

``` java
// A FieldInfo is metadata for a struct field.
type FieldInfo struct {
	Index    []int
	Path     string
	Field    reflect.StructField
	Zero     reflect.Value
	Name     string
	Options  map[string]string
	Embedded bool
	Children []*FieldInfo
	Parent   *FieldInfo
}

// A StructMap is an index of field metadata for a struct.
type StructMap struct {
	Tree  *FieldInfo
	Index []*FieldInfo
	Paths map[string]*FieldInfo
	Names map[string]*FieldInfo
}

// Mapper is a general purpose mapper of names to struct fields.  A Mapper
// behaves like most marshallers in the standard library, obeying a field tag
// for name mapping but also providing a basic transform function.
type Mapper struct {
	cache      map[reflect.Type]*StructMap
	tagName    string
	tagMapFunc func(string) string
	mapFunc    func(string) string
	mutex      sync.Mutex
}
```
Mapper包括StructMap, StructMap包括FieldInfo。


#### StructMap

对于StructMap，如下几个函数都是返回FileInfo的信息，StructMap维护了一个Tree结构，这个index应该是一个tree path，根据这个tree path返回对应的FieldInfo

``` java
// GetByPath returns a *FieldInfo for a given string path.
func (f StructMap) GetByPath(path string) *FieldInfo {
	return f.Paths[path]
}

// GetByTraversal returns a *FieldInfo for a given integer path.  It is
// analogous to reflect.FieldByIndex, but using the cached traversal
// rather than re-executing the reflect machinery each time.
func (f StructMap) GetByTraversal(index []int) *FieldInfo {
	if len(index) == 0 {
		return nil
	}

	tree := f.Tree
	for _, i := range index {
		if i >= len(tree.Children) || tree.Children[i] == nil {
			return nil
		}
		tree = tree.Children[i]
	}
	return tree
}
```


#### Mapper

这里重点看**TypeMap**，他是根据一个Type返回对应的StructMap，流程就是看缓存，如果缓存不命中再看看**getMapping**函数。

``` java
// TypeMap returns a mapping of field strings to int slices representing
// the traversal down the struct to reach the field.
func (m *Mapper) TypeMap(t reflect.Type) *StructMap {
	m.mutex.Lock()
	mapping, ok := m.cache[t]
	if !ok {
		mapping = getMapping(t, m.tagName, m.mapFunc, m.tagMapFunc)
		m.cache[t] = mapping
	}
	m.mutex.Unlock()
	return mapping
}
```

因此这里最重要的函数就是getMapping函数，我们重点看看这个函数


##### getMapping

该函数用到的其他关联的信息如下：
``` java
type typeQueue struct {
	t  reflect.Type
	fi *FieldInfo
	pp string // Parent path
}

type mapf func(string) string

// Deref is Indirect for reflect.Types
func Deref(t reflect.Type) reflect.Type {
	if t.Kind() == reflect.Ptr {
		t = t.Elem()
	}
	return t
}

// parseName parses the tag and the target name for the given field using
// the tagName (eg 'json' for `json:"foo"` tags), mapFunc for mapping the
// field's name to a target name, and tagMapFunc for mapping the tag to
// a target name.
func parseName(field reflect.StructField, tagName string, mapFunc, tagMapFunc mapf) (tag, fieldName string) {
	// first, set the fieldName to the field's name
	fieldName = field.Name
	// if a mapFunc is set, use that to override the fieldName
	if mapFunc != nil {
		fieldName = mapFunc(fieldName)
	}

	// if there's no tag to look for, return the field name
	if tagName == "" {
		return "", fieldName
	}

	// if this tag is not set using the normal convention in the tag,
	// then return the fieldname..  this check is done because according
	// to the reflect documentation:
	//    If the tag does not have the conventional format,
	//    the value returned by Get is unspecified.
	// which doesn't sound great.
	if !strings.Contains(string(field.Tag), tagName+":") {
		return "", fieldName
	}

	// at this point we're fairly sure that we have a tag, so lets pull it out
	tag = field.Tag.Get(tagName)

	// if we have a mapper function, call it on the whole tag
	// XXX: this is a change from the old version, which pulled out the name
	// before the tagMapFunc could be run, but I think this is the right way
	if tagMapFunc != nil {
		tag = tagMapFunc(tag)
	}

	// finally, split the options from the name
	parts := strings.Split(tag, ",")
	fieldName = parts[0]

	return tag, fieldName
}

// parseOptions parses options out of a tag string, skipping the name
func parseOptions(tag string) map[string]string {
	parts := strings.Split(tag, ",")
	options := make(map[string]string, len(parts))
	if len(parts) > 1 {
		for _, opt := range parts[1:] {
			// short circuit potentially expensive split op
			if strings.Contains(opt, "=") {
				kv := strings.Split(opt, "=")
				options[kv[0]] = kv[1]
				continue
			}
			options[opt] = ""
		}
	}
	return options
}

```

getMapping函数是利用一个队列遍历，对从队列中取出来的每个FieldInfo，先判断下他是不是recursive field，然后再取出该FieldInfo所有的field, 对每个Field，调用parseName 对该Field的名字进行处理（使用MapFunc）然后生成新的FieldInfo，
这里调用parseOption函数，该函数会解析tag的值。
如果这个Field是一个内嵌结构体，或者他自身是一个Struct，那么会递归的调用（进入队列），等遍历完全部的Field后，会生成一个StructMap，完成FieldInfo到StructMap的关联。

``` java
func getMapping(t reflect.Type, tagName string, mapFunc, tagMapFunc mapf) *StructMap {
	m := []*FieldInfo{}

	root := &FieldInfo{}
	queue := []typeQueue{}
	queue = append(queue, typeQueue{Deref(t), root, ""})

QueueLoop:
	for len(queue) != 0 {
		// pop the first item off of the queue
		tq := queue[0]
		queue = queue[1:]

		// ignore recursive field
		for p := tq.fi.Parent; p != nil; p = p.Parent {
			if tq.fi.Field.Type == p.Field.Type {
				continue QueueLoop
			}
		}

		nChildren := 0
		if tq.t.Kind() == reflect.Struct {
			nChildren = tq.t.NumField()
		}
		tq.fi.Children = make([]*FieldInfo, nChildren)

		// iterate through all of its fields
		for fieldPos := 0; fieldPos < nChildren; fieldPos++ {

			f := tq.t.Field(fieldPos)

			// parse the tag and the target name using the mapping options for this field
			tag, name := parseName(f, tagName, mapFunc, tagMapFunc)

			// if the name is "-", disabled via a tag, skip it
			if name == "-" {
				continue
			}

			fi := FieldInfo{
				Field:   f,
				Name:    name,
				Zero:    reflect.New(f.Type).Elem(),
				Options: parseOptions(tag),
			}

			// if the path is empty this path is just the name
			if tq.pp == "" {
				fi.Path = fi.Name
			} else {
				fi.Path = tq.pp + "." + fi.Name
			}

			// skip unexported fields
			if len(f.PkgPath) != 0 && !f.Anonymous {
				continue
			}

			// bfs search of anonymous embedded structs
			if f.Anonymous {
				pp := tq.pp
				if tag != "" {
					pp = fi.Path
				}

				fi.Embedded = true
				fi.Index = apnd(tq.fi.Index, fieldPos)
				nChildren := 0
				ft := Deref(f.Type)
				if ft.Kind() == reflect.Struct {
					nChildren = ft.NumField()
				}
				fi.Children = make([]*FieldInfo, nChildren)
				queue = append(queue, typeQueue{Deref(f.Type), &fi, pp})
			} else if fi.Zero.Kind() == reflect.Struct || (fi.Zero.Kind() == reflect.Ptr && fi.Zero.Type().Elem().Kind() == reflect.Struct) {
				fi.Index = apnd(tq.fi.Index, fieldPos)
				fi.Children = make([]*FieldInfo, Deref(f.Type).NumField())
				queue = append(queue, typeQueue{Deref(f.Type), &fi, fi.Path})
			}

			fi.Index = apnd(tq.fi.Index, fieldPos)
			fi.Parent = tq.fi
			tq.fi.Children[fieldPos] = &fi
			m = append(m, &fi)
		}
	}

	flds := &StructMap{Index: m, Tree: root, Paths: map[string]*FieldInfo{}, Names: map[string]*FieldInfo{}}
	for _, fi := range flds.Index {
		flds.Paths[fi.Path] = fi
		if fi.Name != "" && !fi.Embedded {
			flds.Names[fi.Path] = fi
		}
	}

	return flds
}

```


#####  Deref
Deref函数就是返回变量真正的类型：
``` java
func Deref(t reflect.Type) reflect.Type {
		if t.Kind() == reflect.Ptr {
			t = t.Elem()
		}
		return t
 }
```
>当你定义一个名称为 Foo 的结构体，那么它的 kind 是 struct，而它的 type 是 Foo。当使用反射时，我们必须要意识到：在使用 reflect 包时，会假设你清楚的知道自己在做什么，如果使用不当，将会产生 panic。举个例子，你在 int 类型上调用 struct 结构体类型上才用的方法，你的代码就会产生 panic。我们时刻要记住，什么类型有有什么方法可以使用，从而避免产生 panic。如果一个变量是指针、映射、切片、管道、或者数组类型，那么这个变量的类型就可以调用方法 varType.Elem()。Elem returns a type's element type.

##### methodName

methodName函数，该函数是获取调用者的函数名
``` java
// methodName returns the caller of the function calling methodName
func methodName() string {
	pc, _, _, _ := runtime.Caller(2)
	f := runtime.FuncForPC(pc)
	if f == nil {
		return "unknown method"
	}
	return f.Name()
}
```

Callers用来返回调用栈的程序计数器, 放到一个uintptr中。

0 代表 Callers 本身，这和上面的Caller的参数的意义不一样，历史原因造成的。 1 才对应这上面的 0。

比如在上面的例子中增加一个trace函数，被函数Bar调用。

``` java
……
func Bar() {
	fmt.Printf("我是 %s, %s 又在调用我!\n", printMyName(), printCallerName())
	trace()
}
func trace() {
	pc := make([]uintptr, 10) // at least 1 entry needed
	n := runtime.Callers(0, pc)
	for i := 0; i < n; i++ {
		f := runtime.FuncForPC(pc[i])
		file, line := f.FileLine(pc[i])
		fmt.Printf("%s:%d %s\n", file, line, f.Name())
	}
}

/usr/local/go/src/runtime/extern.go:218 runtime.Callers
/Users/yuepan/go/src/git.intra.weibo.com/platform/tool/g/main.go:34 main.trace
/Users/yuepan/go/src/git.intra.weibo.com/platform/tool/g/main.go:20 main.Bar

这见第一个和第二个是要pass的
```


接下来看下Mapper的其他几个函数：

``` java
// FieldMap returns the mapper's mapping of field names to reflect values.  Panics
// if v's Kind is not Struct, or v is not Indirectable to a struct kind.
func (m *Mapper) FieldMap(v reflect.Value) map[string]reflect.Value {
	v = reflect.Indirect(v)
	mustBe(v, reflect.Struct)

	r := map[string]reflect.Value{}
	tm := m.TypeMap(v.Type())
	for tagName, fi := range tm.Names {
		r[tagName] = FieldByIndexes(v, fi.Index)
	}
	return r
}
```
这函数先调用TypeMap获取对应的StructMap，然后调用FieldByIndexes函数获取对应的value，这里我们看看FieldByIndexes函数的实现

###### FieldByIndexes

该函数的indexes就是之前从根到当前节点的index 集合，这里会从root节点开始遍历，如果是struct或者是Map的就给他初始化。然后返回对应的值。
接下来的几个函数有着类似的逻辑。

``` java
func FieldByIndexes(v reflect.Value, indexes []int) reflect.Value {
	for _, i := range indexes {
		v = reflect.Indirect(v).Field(i)
		// if this is a pointer and it's nil, allocate a new value and set it
		if v.Kind() == reflect.Ptr && v.IsNil() {
			alloc := reflect.New(Deref(v.Type()))
			v.Set(alloc)
		}
		if v.Kind() == reflect.Map && v.IsNil() {
			v.Set(reflect.MakeMap(v.Type()))
		}
	}
	return v
}
```

``` java
FieldMap
FieldByName
FieldsByName
```

##### TraversalsByName

TraversalsByName就是根据一个names列表返回对应的index列表，注意这个index是一个二维数组，每个行代表了一个name的index集合。
``` java
// TraversalsByName returns a slice of int slices which represent the struct
// traversals for each mapped name.  Panics if t is not a struct or Indirectable
// to a struct.  Returns empty int slice for each name not found.
func (m *Mapper) TraversalsByName(t reflect.Type, names []string) [][]int {
	r := make([][]int, 0, len(names))
	m.TraversalsByNameFunc(t, names, func(_ int, i []int) error {
		if i == nil {
			r = append(r, []int{})
		} else {
			r = append(r, i)
		}

		return nil
	})
	return r
}
```

##### FieldByIndexesReadOnly
``` java
// FieldByIndexesReadOnly returns a value for a particular struct traversal,
// but is not concerned with allocating nil pointers because the value is
// going to be used for reading and not setting.
func FieldByIndexesReadOnly(v reflect.Value, indexes []int) reflect.Value {
	for _, i := range indexes {
		v = reflect.Indirect(v).Field(i)
	}
	return v
}
```

##### TraversalsByNameFunc
``` java

// TraversalsByNameFunc traverses the mapped names and calls fn with the index of
// each name and the struct traversal represented by that name. Panics if t is not
// a struct or Indirectable to a struct. Returns the first error returned by fn or nil.
func (m *Mapper) TraversalsByNameFunc(t reflect.Type, names []string, fn func(int, []int) error) error {
	t = Deref(t)
	mustBe(t, reflect.Struct)
	tm := m.TypeMap(t)
	for i, name := range names {
		fi, ok := tm.Names[name]
		if !ok {
			if err := fn(i, nil); err != nil {
				return err
			}
		} else {
			if err := fn(i, fi.Index); err != nil {
				return err
			}
		}
	}
	return nil
}

```



