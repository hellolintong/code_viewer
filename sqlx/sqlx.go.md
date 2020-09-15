##### Row

sqlx.Row实现了ColScanner接口, 该接口是对sql的底层封装

```c++
// Row is a reimplementation of sql.Row in order to gain access to the underlying
// sql.Rows.Columns() data, necessary for StructScan.
type Row struct {
	err    error
	unsafe bool
	rows   *sql.Rows
	Mapper *reflectx.Mapper
}
```


```c++
type ColScanner interface {
	Columns() ([]string, error)
	Scan(dest ...interface{}) error
	Err() error
}
```


#### DB 

sql.DB也是对底层DB的封装
``` c++
type DB struct {
	*sql.DB
	driverName string
	unsafe     bool
	Mapper     *reflectx.Mapper
}
```


##### MustExec

MustExec方法实现接口Execer， MustExec方法又调用了MustExec同名函数，该函数根据传入的接口实现代理，这样可以适配多种不同的DB。
MustExec函数就是调用sql.DB的Exec函数

``` c++
func (db *DB) MustExec(query string, args ...interface{}) sql.Result {
	return MustExec(db, query, args...)
}
```


``` c++
type Execer interface {
	Exec(query string, args ...interface{}) (sql.Result, error)
}
```


``` c++
func MustExec(e Execer, query string, args ...interface{}) sql.Result {
	res, err := e.Exec(query, args...)
	if err != nil {
		panic(err)
	}
	return res
}
```




##### Get

先看下Get函数的使用：
``` c++
func queryRowDemo() {
	sqlStr := "select id, name, age from user where id=?"
	var u user
	err := db.Get(&u, sqlStr, 1)
	if err != nil {
		fmt.Printf("get failed, err:%v\n", err)
		return
	}
	fmt.Printf("id:%d name:%s age:%d\n", u.ID, u.Name, u.Age)
}
```

然后看看函数定义，函数的实现逻辑和上文说到的MustExec是一个道理。Get函数先调用底层DB的Queryx函数，进行查询，然后调用scanAny函数填充，这个sacnAny函数涉及到很多reflectx相关的内容。
``` c++
func (db *DB) Get(dest interface{}, query string, args ...interface{}) error {
	return Get(db, dest, query, args...)
}

func Get(q Queryer, dest interface{}, query string, args ...interface{}) error {
	r := q.QueryRowx(query, args...)
	return r.scanAny(dest, false)
}

```

``` c++
func (db *DB) Queryx(query string, args ...interface{}) (*Rows, error) {
	r, err := db.DB.Query(query, args...)
	if err != nil {
		return nil, err
	}
	return &Rows{Rows: r, unsafe: db.unsafe, Mapper: db.Mapper}, err
}
```

##### scanAny

这个函数的作用是把Row的内容填充到struct中。
reflect.ValueOf函数是go的反射初始化一个对象，然后调用Kind()函数，确保是指针，而且不能是Nil。
然后调用[[reflectx.reflect#Deref]]函数, 该函数返回指针对应的底层的类型（实际是调用了Elem函数）
然后调用isScannable函数，该函数判断类型是否是可以scannable的。
如果是可以scannable的，直接调用row的scan函数进行scan。否则要进行比较复杂的处理：调用关联的Mapper的[[reflectx.reflect#TraversalsByName]]函数，该函数返回column对应的fields，然后调用fieldsByTraversal函数该函数中重点调用了[[reflectx.reflect#FieldByIndexes]] 函数，最后将返回的值调用scan函数。
 

``` c++
func (r *Row) scanAny(dest interface{}, structOnly bool) error {
	if r.err != nil {
		return r.err
	}
	if r.rows == nil {
		r.err = sql.ErrNoRows
		return r.err
	}
	defer r.rows.Close()

	v := reflect.ValueOf(dest)
	if v.Kind() != reflect.Ptr {
		return errors.New("must pass a pointer, not a value, to StructScan destination")
	}
	if v.IsNil() {
		return errors.New("nil pointer passed to StructScan destination")
	}

	base := reflectx.Deref(v.Type())
	scannable := isScannable(base)

	if structOnly && scannable {
		return structOnlyError(base)
	}

	columns, err := r.Columns()
	if err != nil {
		return err
	}

	if scannable && len(columns) > 1 {
		return fmt.Errorf("scannable dest type %s with >1 columns (%d) in result", base.Kind(), len(columns))
	}

	if scannable {
		return r.Scan(dest)
	}

	m := r.Mapper

	fields := m.TraversalsByName(v.Type(), columns)
	// if we are not unsafe and are missing fields, return an error
	if f, err := missingFields(fields); err != nil && !r.unsafe {
		return fmt.Errorf("missing destination name %s in %T", columns[f], dest)
	}
	values := make([]interface{}, len(columns))

	err = fieldsByTraversal(v, fields, values, true)
	if err != nil {
		return err
	}
	// scan into the struct field pointers and append to our results
	return r.Scan(values...)
}
```


##### fieldsByTraversal

 ``` c++
func fieldsByTraversal(v reflect.Value, traversals [][]int, values []interface{}, ptrs bool) error {
	v = reflect.Indirect(v)
	if v.Kind() != reflect.Struct {
		return errors.New("argument not a struct")
	}

	for i, traversal := range traversals {
		if len(traversal) == 0 {
			values[i] = new(interface{})
			continue
		}
		f := reflectx.FieldByIndexes(v, traversal)
		if ptrs {
			values[i] = f.Addr().Interface()
		} else {
			values[i] = f.Interface()
		}
	}
	return nil
}
```