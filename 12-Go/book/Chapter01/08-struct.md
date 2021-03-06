
Struct
=========

- 值类型，赋值和传参会复制全部内容。可用 `_` 定义补位字段，支持指向自身类型的指针成员。

```go
type Node struct {
    _    int
    id   int
    data *byte
    next *Node
}

func main() {
    n1 := Node{
        id: 1,
        data: nil,
    }

    n2 := Node{
        id:   2,
        data: nil,
        next: &n1,
    }
}
```

- 隐式初始化(不指定字段名)必须包含全部字段，显式初始化可仅赋值指定字段。

```go
type User struct {
    name string
    age int
}

u1 := User{"Tom", 20}   // 隐式初始化
u2 := User{"Tom"}       // Error: too few values in struct initializer
u3 := User{
    name: "Tom",        // 显式初始化
}
```

- 建议使用显示初始化，隐式初始化当 数据结构改变时，无法编译通过。

```go
type T struct {
    Foo string
    Bar int
}

func main() {
    t := T{"example", 123}             // 隐式初始化
    fmt.Printf("t %+v\n", t)
}
```
更改 T 之后

```go
type T struct {
    Foo string
    Bar int
    Qux string
}

func main() {
    t := T{"example", 123}             // 编译错误
    t := T{Foo: "example", Bar: 123}   // OK
    fmt.Printf("t %+v\n", t)
}
```

- new 关键字初始化，默认初始化为 0。

```go
type Person struct {
    name string
    age int
}

func (p *Person) SetName(name string) {
    p.name = name
}
func (p *Person) SetAge(age int) {
    p.age = age
}

/**
 * 使用 & 创建新的 Person 指针，但是需要初始化值。
 */
func NewPerson1() *Person {
    p := &Person{"Jack", 29}
    return p
}

/**
 * 使用 new 创建新的 Person 指针，默认初始化为 0
 */
func NewPerson2() *Person {
    p := new(Person)
    p.name = "Jack"
    return p
}
```

- 赋值语句如果在一行，最后一个字段后有无逗号均可，如果是多行，每条赋值语句都需以逗号结尾。或者最后一个字段赋值后在同一行加上大括号。

```go
type User struct {
    name string
    age int
}
u1 := User{name: "Tom", age : 20}   // OK
u2 := User{name: "Tom", age : 20,}  // OK
u3 := User{
	name: "Tom",
	age: 20}                    // OK
u4 := User{
	name: "Tom",
	age: 20,                    // OK
}
u5 := User{
	name: "Tom",
	age: 20                     // Error: missing ',' before newline in composite literal
}
```

- 支持匿名结构，可用作结构成员或定义变量。

```go
type File struct {
    name string
    size int
    attr struct {
        perm int
        owner int
    }
}

f := File{
    name: "test.txt",
    size: 1025,
    // attr: {0755, 1},
}

f.attr.owner = 1
f.attr.perm = 0755

var attr = struct {
    perm  int
    owner int
}{2, 0755}

f.attr = attr
```

- 支持 `==`，`!=` 操作符，可用作 map 键值。

```go
type User struct {
    id   int
    name string
}

m := map[User]int{
    User{1, "Tom"}: 100,
}
```

- 可定义字段标签，用反射读取。标签是类型的组成部分。

```go
var u1 struct { name string "username" }
var u2 struct { name string }

u2 = u1   // Error: cannot use u1 (type struct { name string "username" }) as
          // type struct { name string } in assignment
```

- 空结构“节省”内存，比如用来实现 set 数据结构，或者实现没有“状态”只有方法的“静态类”。

```go
var null struct{}
set := make(map[string]struct{})
set["a"] = null
```

- 匿名字段其实就是一个与成员类型同名(不含包名)的字段。被匿名嵌入的可以是任何类型。

```go
type User struct {
    name string
}

type Manager struct {
    User                    // 匿名字段
    title string
}

m := Manager{
    User:  User{"Tom"},     // 匿名字段的显式字段名，与类型名相同
    title: "Administrator",
}
```

- 访问匿名字段方式与普通字段相同，编译器会从外向内逐级查找，直到找到目标或出错。

```go
type Resource struct {
    id int
}

type User struct {
    Resource
    name string
}

type Manager struct {
    User
    title string
}

var m Manager
m.id = 1            // 等价于 m.Resource.id
m.name = "Jack"
m.title = "Administrator"
```

- 外层同名字段会遮蔽内层字段，但相同层次的同名字段编译器会报错，建议使用显示字段名。

```go
type Resource struct {
    id int
    name string
}

type Classify struct {
    id int
}

type User struct {
    Resource                    // Resource.id 与 Classify.id 处于同一层次
    Classify
    name string                 // 遮蔽 Resource.name
}

u := User{
    Resource{1, "people"},
    Classify{100},
    "Jack",
}

println(u.name)                 // User.name: Jack
println(u.Resource.name)        // people
// println(u.id)                // Error: ambiguous selector u.id
println(u.Classify.id)          // 100
```

- 不可同时嵌入某一类型及其指针类型。

```go
type Resource struct {
    id int
}

type User struct {
    *Resource
    // Resource                 // Error: duplicate filed Resource
    name string
}

u := User{
    &Resource{1},
    "Administrator",
}

println(u.id)
println(u.Resource.id)
```

- 面向对象三大特征，Go 仅支持封装，不支持继承，多态。也没有 class 关键字，但可通过 struct 的匿名字段方式，以及Go中的 “方法” 实现，详见 method 一节。

```go
type User struct {
    id   int
    name string
}

type Manager struct {
    User
    title string
}

m := Manager{User{1, "Tom"}, "Administrator"}

// var u User = m // Error: cannot use m (type Manager) as type User in assignment

var u User = m.User // 同类型拷贝

// Go struct 与C struct 内存布局相同，无任何附加的 object 信息
// 上述代码中，变量 m, u 的内存布局：

    |<------ User:24 ------>|<- title:16 ->|
    +--------+--------------+--------------+
m   |  int   |    string    |    string    |
    +--------+--------------+--------------+

       1           Tom        Administrator

    +--------+--------------+
u   |  int   |    string    |
    +--------+--------------+
    |<-id:8->|<- name:16 -->|
```
