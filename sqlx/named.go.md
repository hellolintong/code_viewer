
##### NamedQuery

首先看看NamedQuery的使用方法：
``` java
p2 := &Person{}
rows, err := db.NamedQuery("SELECT * FROM person WHERE first_name=:first_name", p)
```

NamedQuery的函数定义如下，函数调用bindNamedMapper函数，然后调用sql的Query函数
``` java
func NamedQuery(e Ext, query string, arg interface{}) (*Rows, error) {
	q, args, err := bindNamedMapper(BindType(e.DriverName()), query, arg, mapperFor(e))
	if err != nil {
		return nil, err
	}
	return e.Queryx(q, args...)
}
```



##### bindNamedMapper
bindNamedMapper函数定义如下，这里bindNamedMapper根据arg的类型是不是struct的就进入了bindMap和bindStruct这两个函数。
``` java
func bindNamedMapper(bindType int, query string, arg interface{}, m *reflectx.Mapper) (string, []interface{}, error) {
	if maparg, ok := arg.(map[string]interface{}); ok {
		return bindMap(bindType, query, maparg)
	}
	return bindStruct(bindType, query, arg, m)
}
```

#####  bindStruct
首先看看bindStruct函数， 该函数首先调用compileNamedQuery函数，然后我们看看compileNamedQuery函数定义。
``` java
func bindStruct(bindType int, query string, arg interface{}, m *reflectx.Mapper) (string, []interface{}, error) {
	bound, names, err := compileNamedQuery([]byte(query), bindType)
	if err != nil {
		return "", []interface{}{}, err
	}

	arglist, err := bindArgs(names, arg, m)
	if err != nil {
		return "", []interface{}{}, err
	}

	return bound, arglist, nil
}
```

#####  compileNamedQuery

compileNamedQuery函数中的qs参数是要查询的sql语句，从上面的调用例子可见是类似这样的格式：**SELECT \* FROM person WHERE first_name=:first_name**。 该函数的目的是提取出对应的参数并转换为**SELECT \* FROM person WHERE first_name=?** 这种形式。    
	
函数在循环时候，如果发现b 是: 就进入inName状态，然后会收集到name中，直到遇到几种特殊情况（::, :=等），最后是IsOneOf判断是否退出inName状态，然后使用传入的bindType判断要怎么转换，这里可以参考Test的case 知道转换的case有哪些。

最后返回解析后的sql，以及对应的参数列表。


``` java
func compileNamedQuery(qs []byte, bindType int) (query string, names []string, err error) {
	names = make([]string, 0, 10)
	rebound := make([]byte, 0, len(qs))

	inName := false
	last := len(qs) - 1
	currentVar := 1
	name := make([]byte, 0, 10)

	for i, b := range qs {
		// a ':' while we're in a name is an error
		if b == ':' {
			// if this is the second ':' in a '::' escape sequence, append a ':'
			if inName && i > 0 && qs[i-1] == ':' {
				rebound = append(rebound, ':')
				inName = false
				continue
			} else if inName {
				err = errors.New("unexpected `:` while reading named param at " + strconv.Itoa(i))
				return query, names, err
			}
			inName = true
			name = []byte{}
		} else if inName && i > 0 && b == '=' {
			rebound = append(rebound, ':', '=')
			inName = false
			continue
			// if we're in a name, and this is an allowed character, continue
		} else if inName && (unicode.IsOneOf(allowedBindRunes, rune(b)) || b == '_' || b == '.') && i != last {
			// append the byte to the name if we are in a name and not on the last byte
			name = append(name, b)
			// if we're in a name and it's not an allowed character, the name is done
		} else if inName {
			inName = false
			// if this is the final byte of the string and it is part of the name, then
			// make sure to add it to the name
			if i == last && unicode.IsOneOf(allowedBindRunes, rune(b)) {
				name = append(name, b)
			}
			// add the string representation to the names list
			names = append(names, string(name))
			// add a proper bindvar for the bindType
			switch bindType {
			// oracle only supports named type bind vars even for positional
			case NAMED:
				rebound = append(rebound, ':')
				rebound = append(rebound, name...)
			case QUESTION, UNKNOWN:
				rebound = append(rebound, '?')
			case DOLLAR:
				rebound = append(rebound, '$')
				for _, b := range strconv.Itoa(currentVar) {
					rebound = append(rebound, byte(b))
				}
				currentVar++
			case AT:
				rebound = append(rebound, '@', 'p')
				for _, b := range strconv.Itoa(currentVar) {
					rebound = append(rebound, byte(b))
				}
				currentVar++
			}
			// add this byte to string unless it was not part of the name
			if i != last {
				rebound = append(rebound, b)
			} else if !unicode.IsOneOf(allowedBindRunes, rune(b)) {
				rebound = append(rebound, b)
			}
		} else {
			// this is a normal byte and should just go onto the rebound query
			rebound = append(rebound, b)
		}
	}

	return string(rebound), names, err
}
```

``` java
var allowedBindRunes = []*unicode.RangeTable{unicode.Letter, unicode.Digit}
```

``` java
{
			Q: `INSERT INTO foo (a,b,c,d) VALUES (:name, :age, :first, :last)`,
			R: `INSERT INTO foo (a,b,c,d) VALUES (?, ?, ?, ?)`,
			D: `INSERT INTO foo (a,b,c,d) VALUES ($1, $2, $3, $4)`,
			T: `INSERT INTO foo (a,b,c,d) VALUES (@p1, @p2, @p3, @p4)`,
			N: `INSERT INTO foo (a,b,c,d) VALUES (:name, :age, :first, :last)`,
			V: []string{"name", "age", "first", "last"},
},
```


##### bindArgs

这里的arg是类似p这种struct，然后这里会调用 [[reflectx.reflect#TraversalsByNameFunc]] 在调用中，使用[[reflectx.reflect#FieldByIndexesReadOnly]] 拿到name对应的真正的value列表。最后就返回sql和对应的参数给底层原生的sql执行。



``` java
func bindArgs(names []string, arg interface{}, m *reflectx.Mapper) ([]interface{}, error) {
	arglist := make([]interface{}, 0, len(names))

	// grab the indirected value of arg
	v := reflect.ValueOf(arg)
	for v = reflect.ValueOf(arg); v.Kind() == reflect.Ptr; {
		v = v.Elem()
	}

	err := m.TraversalsByNameFunc(v.Type(), names, func(i int, t []int) error {
		if len(t) == 0 {
			return fmt.Errorf("could not find name %s in %#v", names[i], arg)
		}

		val := reflectx.FieldByIndexesReadOnly(v, t)
		arglist = append(arglist, val.Interface())

		return nil
	})

	return arglist, err
}
```