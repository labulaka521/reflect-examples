<p align="center">
<img 
    src="img/logo.png"
    width="500" alt="Golang reflect package examples">
<br/><br/>
</p>

这个仓库包含了`reflect`包的示例
主要用于编码解码和动态调用函数  
这些例子大多来自于我过去从事的项目，也有一些来自我目前正在从事的项目
您还将在示例中找到有用的注释，这将帮助您理解代码。
有些是我的，有些是从godoc网站上下载的。

如果你想贡献这个仓库，不要犹豫创建一个PR吧
gopher的logo来自[@egonelbre/gophers](https://github.com/egonelbre/gophers).


### 目录
- [读取struct的tag](#读取struct的tag)
- [获取并设置struct字段](#获取并设置struct字段)
- [使用字符串填充切片而不知道它的类型](#使用字符串填充切片而不知道它的类型)
- [用值填充切片](#用值填充切片)
- [设置一个数字的值](#设置一个数字的值)
- [将键值对解码为map](#将键值对解码为map)
- [解码键值对到struct](#解码键值对到struct)
- [将结构体编码为键值对](#将结构体编码为键值对)
- [检查底层类型是否实现接口](#检查底层类型是否实现接口)
- [使用指针包装`reflect.Value`(`T`=>`*T`)](#使用指针包装`reflect.Value`(`T`=>`*T`))
- [函数调用](#函数调用)
  - [调用没有参数的方法也没有返回值](#调用没有参数的方法也没有返回值)
  - [调用含有参数的方法](#调用含有参数的方法)
  - [调用带有参数的函数验证返回值](#调用带有参数的函数验证返回值)
  - [动态调用一个函数类似于template/text包](#调用一个函数类似于template/text包)
  - [调用可变参数的函数](#调用可变参数的函数)
  - [运行时创建一个函数](#运行时创建一个函数)


### 读取struct的tag

```go
package main

import (
	"fmt"
	"reflect"
)

type User struct {
	Email  string `mcl:"email"`
	Name   string `mcl:"name"`
	Age    int    `mcl:"age"`
	Github string `mcl:"github" default:"a8m"`
}

func main() {
	var u interface{} = User{}
	// TypeOf返回代表u的动态类型的反射Type
	t := reflect.TypeOf(u)
	if t.Kind() != reflect.Struct {
		return
	}
	for i := 0; i < t.NumField(); i++ {
		f := t.Field(i)
		fmt.Println(f.Tag.Get("mcl"), f.Tag.Get("default"))
	}
}
//输出  
// email   
// name   
// age   
// github a8m 
```

 
调用`Kind()`返回的是这个列表中定义的值[Kind()](https://golang.org/pkg/reflect/#Kind).  
[Playground](https://play.studygolang.com/p/yF7vdNiNPmh)

### 获取和设置结构体字段
```go
package main

import (
	"fmt"
	"reflect"
)

type User struct {
	Email  string `mcl:"email"`
	Name   string `mcl:"name"`
	Age    int    `mcl:"age"`
	Github string `mcl:"github" default:"a8m"`
}

func main() {
	u := &User{Name: "Ariel Mashraki"}
	// Elem返回指向u的指针
	v := reflect.ValueOf(u).Elem()
	f := v.FieldByName("Github")
	// 确保这个字段被定义，并且可以被设置
	if !f.IsValid() || !f.CanSet() {
		return
	}
	if f.Kind() != reflect.String || f.String() != "" {
		return
	}
	// 设置字段的值
	f.SetString("a8m")
	fmt.Printf("Github username was changed to: %q\n",u.Github)
}
//输出  
//Github username was changed to: "a8m"
```

[Playground](https://play.studygolang.com/p/OCvqjWEtTZa)

### 使用字符串填充切片而不知道它的类型

```go
package main

import (
	"fmt"
	"io"
	"reflect"
)

func main() {
	var (
		a []string
		b []interface{}
		c []io.Writer
	)
	fmt.Println(fill(&a), a) // 通过
	fmt.Println(fill(&b), b) // 通过
	fmt.Println(fill(&c), c) // 失败

}

func fill(i interface{}) error {
	v := reflect.ValueOf(i)
	// 判断是否为指针
	if v.Kind() != reflect.Ptr {
		return fmt.Errorf("non-pointer %v", v.Type())
	}
	// 获取指针v指向的值
	v = v.Elem()
	// 判断值的类型是否为slice
	if v.Kind() != reflect.Slice {
		return fmt.Errorf("can not fill non-slice value")
	}
	v.Set(reflect.MakeSlice(v.Type(), 3, 3))
	// 验证切片的类型
	if !canAssign(v.Index(0)) {
		return fmt.Errorf("can not assign string to slice elements")
	}
	for i, w := range []string{"foo", "bar", "baz"} {
		v.Index(i).Set(reflect.ValueOf(w))
	}
	return nil
}

// 接收一个string或一个空的interface
func canAssign(v reflect.Value) bool {
	return v.Kind() == reflect.String || (v.Kind() == reflect.Interface && v.NumMethod() == 0)
}

// 输出
//<nil> [foo bar baz]   
//<nil> [foo bar baz]          
//can not assign string to slice elements [<nil> <nil> <nil>]
```
[Playground](https://play.studygolang.com/p/RCvfjdYA1K8)

 
 
### 设置一个数字的值
```go
package main

import (
	"fmt"
	"reflect"
)

const n = 255

func main() {
	var (
		a int8
		b int16
		c uint
		d float32
		e string
	)
	fmt.Println(fill(&a), a)
	fmt.Println(fill(&b), b)
	fmt.Println(fill(&c), c)
	fmt.Println(fill(&d), c)
	fmt.Println(fill(&e), e)
}

func fill(i interface{}) error {
	v := reflect.ValueOf(i)
	if v.Kind() != reflect.Ptr {
		return fmt.Errorf("non-pointer %v", v.Type())
	}
	// get the value that the pointer v points to.
	v = v.Elem()
	switch v.Kind() {
	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
		if v.OverflowInt(n) {
			return fmt.Errorf("can't assign value due to %s-overflow", v.Kind())
		}
		v.SetInt(n)
	case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64, reflect.Uintptr:
		if v.OverflowUint(n) {
			return fmt.Errorf("can't assign value due to %s-overflow", v.Kind())
		}
		v.SetUint(n)
	case reflect.Float32, reflect.Float64:
		if v.OverflowFloat(n) {
			return fmt.Errorf("can't assign value due to %s-overflow", v.Kind())
		}
		v.SetFloat(n)
	default:
			return fmt.Errorf("can't assign value to a non-number type")
	}
	return nil
}

// 输出
//can not assign value due to int8 overflow 0
//<nil> 255
//<nil> 255
//<nil> 255
//can't assign value to a non-number type 
```
[Playground](https://play.studygolang.com/p/BPKoEp_CcVD)

### 将键值对解码为map

```go
package main

import (
	"fmt"
	"reflect"
	"strconv"
	"strings"
)

func main() {
	var (
		m0 = make(map[int]string)
		m1 = make(map[int64]string)
		m2 map[interface{}]string
		m3 map[interface{}]interface{}
		m4 map[bool]string
	)
	s := "1=foo,2=bar,3=baz"

	fmt.Println(decodeMap(s, &m0), m0) // pass
	fmt.Println(decodeMap(s, &m1), m1) // pass
	fmt.Println(decodeMap(s, &m2), m2) // pass
	fmt.Println(decodeMap(s, &m3), m3) // pass
	fmt.Println(decodeMap(s, &m4), m4) // fail
}
func decodeMap(s string, i interface{}) error {
	v := reflect.ValueOf(i)
	if v.Kind() != reflect.Ptr {
		return fmt.Errorf("non-pointer %v", v.Type())
	}
	// 获取指针v指向的值
	v = v.Elem()
	t := v.Type()
	// 如果为nil 分配新的map m2,m3,m4
	if v.IsNil() {
		v.Set(reflect.MakeMap(t))
	}
	for _, kv := range strings.Split(s, ",") {
		s := strings.Split(kv, "=")
		n, err := strconv.Atoi(s[0])
		if err != nil {
			return fmt.Errorf("failed to parse number: %v", err)
		}
		k, e := reflect.ValueOf(n), reflect.ValueOf(s[1])
		// 获取map key的类型 int
		kt := t.Key()

		if !k.Type().ConvertibleTo(kt) {
			return fmt.Errorf("can not convert key to type %v", kt.Kind())
		}
		k = k.Convert(kt)
		// 获取map value的类型
		et := t.Elem()

		if et.Kind() != v.Kind() && !e.Type().ConvertibleTo(et) {
			return fmt.Errorf("can not assign value to type %v", kt.Kind())
		}
		v.SetMapIndex(k, e.Convert(et))

	}
	return nil

}
// 输出
//<nil> map[1:foo 2:bar 3:baz]
//<nil> map[1:foo 2:bar 3:baz]
//<nil> map[1:foo 2:bar 3:baz]
//<nil> map[1:foo 2:bar 3:baz]
// can not convert key to type bool map[]
```
[Playground](https://play.studygolang.com/p/zA5tqcVpf8m)

### 解码键值对到struct
```go
package main

import (
	"fmt"
	"reflect"
	"strings"
)

type User struct {
	Name    string
	Github  string
	private string
}

func main() {
	var (
		v0 User
		v1 *User
		v2 = new(User)
		v3 struct{ Name string }
	)
	s := "Name=Ariel,Github=a8m"

	fmt.Println(decodeStruct(s, &v0), v0) // pass
	fmt.Println(decodeStruct(s, v1), v1)  // fail
	fmt.Println(decodeStruct(s, v2), v2)  // pass
	fmt.Println(decodeStruct(s, v3), v3)  // fail
	fmt.Println(decodeStruct(s, &v3), v3) // pass
}
func decodeStruct(s string, i interface{}) error {
	v := reflect.ValueOf(i)
	// 判断是否是指针或者为nil
	if v.Kind() != reflect.Ptr || v.IsNil() {
		return fmt.Errorf("decode requires non-nil pointer")
	}
	// 获取指针v指向的值
	v = v.Elem()
	for _, kv := range strings.Split(s, ",") {
		s := strings.Split(kv, "=")
		f := v.FieldByName(s[0])
		//确保字段被定义 并且可以被修改
		if !f.IsValid() || !f.CanSet() {
			continue
		}
		f.SetString(s[1])
	}
	return nil
}

// 输出
//<nil> {Ariel a8m }
//decode requires non-nil pointer <nil>
//<nil> &{Ariel a8m }
//decode requires non-nil pointer {}
//<nil> {Ariel}
```
[Playground](https://play.studygolang.com/p/PQIkMHbODf_P)

### 将结构体编码为键值对
```go
package main

import (
	"fmt"
	"reflect"
	"strings"
)

type User struct {
	Email   string `kv:"email,omitempty"`
	Name    string `kv:"name,omitempty"`
	Github  string `kv:"github,omitempty"`
	private string
}

func main() {
	var (
		u = User{Name: "Ariel", Github: "a8m"}
		v = struct {
			A, B, C string
		}{
			"foo",
			"bar",
			"baz",
		}
		w = &User{}
	)
	fmt.Println(encode(u))
	fmt.Println(encode(v))
	fmt.Println(encode(w))
}

// 只支持struct 并且字段类型是string
func encode(i interface{}) (string, error) {
	v := reflect.ValueOf(i)
	t := v.Type()
	if t.Kind() != reflect.Struct {
		return "", fmt.Errorf("type %s is not supported", t.Kind())
	}
	var s []string
	for i := 0; i < t.NumField(); i++ {
		f := t.Field(i)
		//跳过未导出的字段。从godoc:
		// PkgPath是限定小写(未导出)的包路径
		//字段名。它对于大写(导出)字段名是空的。
		if f.PkgPath != "" {
			continue
		}
		fv := v.Field(i)
		key, omit := readTag(f)
		// 如果值为空并且设置了omitempty就跳过
		if omit && fv.String() == "" {
			continue
		}
		s = append(s, fmt.Sprintf("%s=%s", key, fv.String()))

	}
	return strings.Join(s, ","), nil
}

func readTag(f reflect.StructField) (string, bool) {
	val, ok := f.Tag.Lookup("kv")
	if !ok {
		return f.Name, false
	}
	opts := strings.Split(val, ",")
	var omit bool

	if len(opts) == 2 {
		omit = opts[1] == "omitempty"
	}
	return opts[0], omit
}

//输出
//name=Ariel,github=a8m <nil>
//A=foo,B=bar,C=baz <nil>
//type ptr is not supported
```
[Playground](https://play.studygolang.com/p/iMf3Pc07kLj)

### 检查底层类型是否实现接口
```go
package main

import (
	"fmt"
	"reflect"
)

type User struct {
	Email   string `kv:"email,omitempty"`
	Name    string `kv:"name,omitempty"`
	Github  string `kv:"github,omitempty"`
	private string
}
type Marshaler interface {
	MarshalKV() (string, error)
}

func (u *User) MarshalKV() (string, error) {
	return fmt.Sprintf("name=%s,email=%s,github=%s", u.Name, u.Email, u.Github), nil
}

func main() {
	
	fmt.Println(encode(User{"boring", "Ariel", "a8m", ""}))	// 未实现接口
	fmt.Println(encode(&User{Github: "posener", Name: "Eyal", Email: "boring"}))
}

var marshalertype = reflect.TypeOf(new(Marshaler)).Elem()

func encode(i interface{}) (string, error) {
	t := reflect.TypeOf(i)
	if !t.Implements(marshalertype) {
		return "", fmt.Errorf("encode only supports structs that implement the Marshaler interface")
	}
	m, _ := reflect.ValueOf(i).Interface().(Marshaler)
	return m.MarshalKV()
}

// 输出
// encode only supports structs that implement the Marshaler interface
//name=Eyal,email=boring,github=posener <nil>
```
[Playground](https://play.studygolang.com/p/qbJctpjXaX4)

### 使用指针包装`reflect.Value`(`T`=>`*T`)
```go
package main

import (
	"fmt"
	"reflect"
)

type User struct {
	Name string
}

func main() {
	// T => *T
	u1 := User{"a8m"}
	p1 := ptr(reflect.ValueOf(u1))
	fmt.Println(u1 == p1.Elem().Interface())

	// *T => **T
	u2 := &User{"a8m"}
	p2 := ptr(reflect.ValueOf(u2))
	fmt.Println(*u2 == p2.Elem().Elem().Interface())
}

// ptr wraps the given value with pointer: V => *V, *V => **V, etc.
func ptr(v reflect.Value) reflect.Value {
	pt := reflect.PtrTo(v.Type()) // create a *T type.
	pv := reflect.New(pt.Elem())  // create a reflect.Value of type *T.
	pv.Elem().Set(v)              // sets pv to point to underlying value of v.
	return pv
}
```
[Playground](https://play.studygolang.com/p/JJ8ii8_TQHm)


### 函数调用

#### 调用没有参数的方法也没有返回值
```go
package main

import (
	"fmt"
	"reflect"
)

type A struct{}

func (a A) Hello() {
	fmt.Println("hello word")
}
func main() {
	// ValueOf返回一个新值，这是Go的反射接口
	v := reflect.ValueOf(A{})
	m := v.MethodByName("Hello")
	if m.Kind() != reflect.Func {
		return
	}
	m.Call(nil)

}
```
[Playground](https://play.studygolang.com/p/xK5g7fBPmm_o)

#### 调用含有参数的方法
```go
package main

import (
	"fmt"
	"reflect"
)

type A struct{}

func (a A) Hello(word string) {
	fmt.Println("hello", word)
}
func main() {
	// ValueOf返回一个新值，这是Go的反射接口
	v := reflect.ValueOf(A{})
	m := v.MethodByName("Hello")
	if m.Kind() != reflect.Func {
		return
	}
	fmt.Println(m.Type().In(0))
	m.Call([]reflect.Value{reflect.ValueOf("word")})
}
```
[Playground](https://play.studygolang.com/p/fWPhftoI1Cy)

#### 调用带有参数的函数验证返回值
```go
package main

import (
	"fmt"
	"reflect"
)

func Add(a, b int) int { return a + b }
func main() {
	v := reflect.ValueOf(Add)
	if v.Kind() != reflect.Func {
		return
	}

	t := v.Type()
	argv := make([]reflect.Value, t.NumIn())
	for i := range argv {
		// 验证参数i的类型
		if t.In(i).Kind() != reflect.Int {
			return
		}
		argv[i] = reflect.ValueOf(i)
	}
	// 调用函数

	result := v.Call(argv)

	if len(result) != 1 || result[0].Kind() != reflect.Int {
		return
	}
	fmt.Println(result[0].Int())

}
// 输出
// 1
```
[Playground](https://play.studygolang.com/p/vsLqLHlu5nv)

#### 动态调用一个函数类似于template/text包
```go
package main

import (
	"fmt"
	"html/template"
	"reflect"
	"strconv"
	"strings"
)

func main() {
	funcs := template.FuncMap{
		"trim":    strings.Trim,
		"lower":   strings.ToLower,
		"repeat":  strings.Repeat,
		"replace": strings.Replace,
	}
	fmt.Println(eval(`{{ "hello" 4 | repeat }}`, funcs))
	fmt.Println(eval(`{{ "Hello-World" | lower }}`, funcs))
	fmt.Println(eval(`{{ "foobarfoo" "foo" "bar" -1 | replace }}`, funcs))
	fmt.Println(eval(`{{ "-^-Hello-^-" "-^" | trim }}`, funcs))
}

// evaluate an expression. note that this implemetation is assuming that the
// input is valid, and also very limited. for example, whitespaces are not allowed
// inside a quoted string.
func eval(s string, funcs template.FuncMap) (string, error) {
	args, name := parseArgs(s)
	fn, ok := funcs[name]
	if !ok {
		return "", fmt.Errorf("function %s is not defined", name)
	}
	v := reflect.ValueOf(fn)
	t := v.Type()
	if len(args) != t.NumIn() {
		return "", fmt.Errorf("invalid number of arguments. got: %v, want: %v", len(args), t.NumIn())
	}
	argv := make([]reflect.Value, len(args))
	// go over the arguments, validate and build them.
	// note that we support only int, and string in this simple example.
	for i := range argv {
		var argType reflect.Kind
		// if the argument "i" is string.
		if strings.HasPrefix(args[i], "\"") {
			argType = reflect.String
			argv[i] = reflect.ValueOf(strings.Trim(args[i], "\""))
		} else {
			argType = reflect.Int
			// assume that the input is valid.
			num, _ := strconv.Atoi(args[i])
			argv[i] = reflect.ValueOf(num)
		}
		if t.In(i).Kind() != argType {
			return "", fmt.Errorf("Invalid argument. got: %v, want: %v", argType, t.In(i).Kind())
		}
	}
	result := v.Call(argv)
	// in real-world code, we validate it before executing the function,
	// using the v.NumOut() method. similiar to the text/template package.
	if len(result) != 1 || result[0].Kind() != reflect.String {
		return "", fmt.Errorf("function %s must return a one string value", name)
	}
	return result[0].String(), nil
}

// parseArgs is an auxiliary function, that extract the function and its
// parameter from the given expression.
func parseArgs(s string) ([]string, string) {
	args := strings.Split(strings.Trim(s, "{ }"), "|")
	return strings.Split(strings.Trim(args[0], " "), " "), strings.Trim(args[len(args)-1], " ")
}
```

#### 调用可变参数的函数
```go
package main

import (
	"fmt"
	"math/rand"
	"reflect"
)

func Sum(x1, x2 int, xs ...int) int {
	sum := x1 + x2
	for _, xi := range xs {
		sum += xi
	}
	return sum
}

func main() {
	v := reflect.ValueOf(Sum)
	if v.Kind() != reflect.Func {
		return
	}
	t := v.Type()
	// 获取参数个数
	argc := t.NumIn()
	
	// 判断是否为可变参数
	if t.IsVariadic() {
		argc += rand.Intn(10)
	}
	
	argv := make([]reflect.Value, argc)
	for i := range argv {
		argv[i] = reflect.ValueOf(i)
	}
	// 调用函数 0 1 2 3 
	result := v.Call(argv)
	fmt.Println(result[0].Int())
}

// 输出
// 6
```
[Playground](https://play.studygolang.com/p/cJXMxfzymBH)

#### 运行时创建一个函数
```go
package main

import (
	"fmt"
	"reflect"
)

type Add func(int64, int64) int64

func main() {
	t := reflect.TypeOf(Add(nil))
	mul := reflect.MakeFunc(t, func(args []reflect.Value) []reflect.Value {
		a := args[0].Int()
		b := args[1].Int()
		return []reflect.Value{reflect.ValueOf(a+b)}
	})
	fn, ok := mul.Interface().(Add)
	if !ok {
		return
	}
	fmt.Println(fn(2,3))
}
```
[Playground](https://play.studygolang.com/p/oSvEk8QzTTL)
