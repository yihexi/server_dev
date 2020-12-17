#### 启用CGO

```go
package main
import "C"
func main() { 
  println("hello cgo") 
}
```

只要引入了import "C"，就启用了CGO



#### 使用C字符串

```go
package main
//#include <stdio.h> 
import "C"

func main() { 
  C.puts(C.CString("Hello, World\n")) 
}
```

`"Hello, World\n"`是Go字符串，通过`C.CString` 变成了C字符串。在Go的编译环境下没法写C字符串字面量，只能通过`C.CString`转。

Go环境下调用C函数也是通过`C.puts`。



#### 使用自己的C函数

```go
package main

/* #include <stdio.h>

static void SayHello(const char* s) { puts(s); } 
*/ 
import "C"

func main() { 
  C.SayHello(C.CString("Hello, World\n")) 
}
```

`C.SayHello`调用自己写的C函数，自己写的C函数通过注释引入的。



#### C和Go代码分离

先写一个C文件hello.c

```c
// hello.c

#include <stdio.h>

void SayHello(const char* s) { puts(s); }
```

然后Go文件中引入`SayHello`函数

```go
package main

//void SayHello(const char* s); 
import "C"

func main() {
  C.SayHello(C.CString("Hello, World\n")) 
}
```



#### 通过C接口导出C++实现的函数

头文件hello.h

```c
// hello.h 
void SayHello(const char* s);
```

函数的C++实现

```c++
// hello.cpp

#include <iostream>

extern "C" { 
  #include "hello.h" 
}

void SayHello(const char* s) { 
  std::cout << s; 
}
```

Go使用：

```go
package main

//void SayHello(const char* s); 
import "C"

func main() {
  C.SayHello(C.CString("Hello, World\n")) 
}
```



#### 导出Go语言函数给C语言用

头文件

```c
// hello.h 
void SayHello(/*const*/ char* s);
```

`SayHello`的Go语言实现

``` go
// hello.go package main

import "C"

import "fmt"

//export SayHello 
func SayHello(s *C.char) { 
  fmt.Print(C.GoString(s)) 
}
```

在C中使用`SayHello`就和使用一般C函数一样，引入头文件，然后使用。在Go中使用`SayHello`和之前用C实现`SayHello`也一样。头文件`hello.h`隔离的实现的语言。



#### \_\_GoString\_\_类型

\_\_GoString\_\_是CGO中定义的C语言类型，由Go1.10引进：

```go
// +build go1.10
package main

//void SayHello(_GoString_ s); 
import "C"
import "fmt"

func main() { 
  C.SayHello("Hello, World\n") 
}

//export SayHello 
func SayHello(s string) { 
  fmt.Print(s) 
}
```

首先`SayHello`完全是一个Go函数，参数也是Go的string类型。导出成一个C函数，C函数的参数是\_\_GoString\_\_类型，这是一个C语言的类型，在编译的时候可以和Go语言的字符串类型无缝转换。



#### CGO中的辅助函数困境

比如有一个cgo_helper的包

```go
package cgo_helper

//#include <stdio.h> import "C"

type CChar C.char

func (p *CChar) GoString() string { 
  return C.GoString((*C.char)(p)) 
}

func PrintCString(cs *C.char) { 
  C.puts(cs) 
}
```

在main中使用：

```go
package main

//static const char* cs = "hello";

import "C" import "./cgo_helper"

func main() { 
  cgo_helper.PrintCString(C.cs) 
}
```

这样是不行的，因为变量cs的类型是`main.C.char`，而`cgo_helper.PrintCString`函数的参数类型是`cgo_helper.C.char`，不同的Go函数包，引入C之后，生成不同的虚拟"C"包，其中的C类型不同。因此辅助函数只能写在一个包中。



#### 编译链接参数

```go
// #cgo CFLAGS: -DPNG_DEBUG=1 -I./include 
// #cgo LDFLAGS: -L/usr/local/lib -lpng 
// #include <png.h> 
import "C"
```

上述代码的意思是：

* 定义一个`PNG_DEBUG`宏
* 包含头文件目录`./include`
* 链接时寻找路径`/usr/local/lib`
* 链接库`-lpng`

C头文件检索目录可以是相对目录，但是库文件检索目录则需要绝对路径。

`${SRCDIR}`当前包的路径。

条件编译

```go
package main

/* 
#cgo windows CFLAGS: -DCGO_OS_WINDOWS=1 
#cgo darwin CFLAGS: -DCGO_OS_DARWIN=1 
#cgo linux CFLAGS: -DCGO_OS_LINUX=1

#if defined(CGO_OS_WINDOWS)
	static const char* os = "windows";
#elif defined(CGO_OS_DARWIN)
	static const char* os = "darwin";
#elif defined(CGO_OS_LINUX)
	static const char* os = "linux";
#else 
# error(unknown os) #endif */ 
import "C"
func main() { 
  print(C.GoString(C.os)) 
}
```



#### 数值类型转换

| C语言类型              | CGO类型     | Go语言类型 |
| ---------------------- | ----------- | ---------- |
| char                   | C.char      | byte       |
| singed char            | C.schar     | int8       |
| unsigned char          | C.uchar     | uint8      |
| short                  | C.short     | int16      |
| unsigned short         | C.ushort    | uint16     |
| int                    | C.int       | int32      |
| unsigned int           | C.uint      | uint32     |
| long                   | C.long      | int32      |
| unsigned long          | C.ulong     | uint32     |
| long long int          | C.longlong  | int64      |
| unsigned long long int | C.ulonglong | uint64     |
| float                  | C.float     | float32    |
| double                 | C.double    | float64    |
| size_t                 | C.size_t    | uint       |

虽然在C语言中 int 、 short 等类型没有明确定义内存大小，但是在CGO中它们的内存大小是确定的。在CGO中，C语言的 int 和 long 类型都 是对应4个字节的内存大小。



#### 字符串和切片类型转换

```c
typedef struct { const char *p; GoInt n; } GoString; 
typedef void *GoMap; 
typedef void *GoChan; 
typedef struct { void *t; void *v; } GoInterface; 
typedef struct { void *data; GoInt len; GoInt cap; } GoSlice;
```

只有字符串和切片有使用价值，因为CGO提供了相关的辅助函数。

```go
// Go string to C string 
// The C string is allocated in the C heap using malloc.
// It is the caller's responsibility to arrange for it to be 
// freed, such as by calling C.free (be sure to include stdlib.h 
// if C.free is needed).
func C.CString(string) *C.char

// Go []byte slice to C array 
// The C array is allocated in the C heap using malloc.
// It is the caller's responsibility to arrange for it to be
// freed, such as by calling C.free (be sure to include stdlib.h 
// if C.free is needed).
func C.CBytes([]byte) unsafe.Pointer

// C string to Go string 
func C.GoString(*C.char) string

// C data with explicit length to Go string 
func C.GoStringN(*C.char, C.int) string

// C data with explicit length to Go []byte 
func C.GoBytes(unsafe.Pointer, C.int) []byte
```

该组辅助函数都是以克隆的方式运行。当Go语言字符串和切片向C语言转换时，克 隆的内存由C语言的 malloc 函数分配，最终可以通过 free 函数释放。当C语言 字符串或数组向Go语言转换时，克隆的内存由Go语言分配管理。



#### 结构体、联合、枚举类型转换

```go
/* 
struct A {
	int i;
	float f; 
}; */ 
import "C" 
import "fmt"

func main() { 
  var a C.struct_A 
  fmt.Println(a.i) 
  fmt.Println(a.f) 
}
```

`C.struct_A `表示引用了虚拟C包中的A结构体。

如果结构体的成员名字中碰巧是Go语言的关键字，可以通过在成员名开头添加下划线来访问。

不能访问：

* 对于指定了特殊对齐规则的结构体，无法在CGO中访问。

* C语言结构体中位字段对应的成员无法在Go语言中访问，需要使用辅助函数。

* 对应零长数组的成员，无法在Go语 言中直接访问数组的元素，但其中零长的数组成员所在位置的偏移量依然可以通 过 unsafe.Offsetof(a.arr) 来访问。

* 在C语言中，我们无法直接访问Go语言定义的结构体类型。

* 对于联合类型，我们可以通过 C.union_xxx 来访问C语言中定义的 union xxx 类型。但是Go语言中并不支持C语言联合类型，它们会被转为对应大小的字节 数组。



#### Go直接访问C内存方式

```go
/* 
static char arr[10];
static char *s = "Hello";
*/ 
import "C" import "fmt"

func main() {
  // 通过 reflect.SliceHeader 转换 
  var arr0 []byte 
  var arr0Hdr = (*reflect.SliceHeader)(unsafe.Pointer(&arr0))
  arr0Hdr.Data = uintptr(unsafe.Pointer(&C.arr[0]))
	arr0Hdr.Len = 10 
  arr0Hdr.Cap = 10
  
  // 通过切片语法转换 
  arr1 := (*[31]byte)(unsafe.Pointer(&C.arr[0]))[:10:10]
  
  var s0 string 
  var s0Hdr := (*reflect.StringHeader)(unsafe.Pointer(&s0)) 
  s0Hdr.Data = uintptr(unsafe.Pointer(C.s)) 
  s0Hdr.Len = int(C.strlen(C.s))
  
  sLen := int(C.strlen(C.s)) 
  s1 := string((*[31]byte)(unsafe.Pointer(&C.s[0]))[:sLen:sLen]) 
}
```

第一段是直接让`arr0`的data指向C类型的arr数组，然后手动设置下大小。第二种方式是拿到`C.arr`的内存，强行转成一个长度为31的byte数组指针，然后截取前10个byte`[0:10:10]`生成slice。

说明长度为31的byte数组也是一块裸内存？

下边对string的处理方式和byte数组的处理方式是一致的。

如果不copy内存，需要保证在go使用期间，底层的内存不会被C语言释放。并且不发生变化？



#### 指针类型转换

在Go语言中两个指针的类型完全一致则不需要转换可以直接通用。如果一个指针类 型是用type命令在另一个指针类型基础之上构建的，换言之两个指针底层是相同完 全结构的指针，那么我我们可以通过直接强制转换语法进行指针间的转换。但是 cgo经常要面对的是2个完全不同类型的指针间的转换，原则上这种操作在纯Go语 言代码是严格禁止的。

CGO可以打破这个禁忌。

```go
var p *X 
var q *Y

q = (*Y)(unsafe.Pointer(p)) // *X => *Y 
p = (*X)(unsafe.Pointer(q)) // *Y => *X
```



#### <errno.h>

C语言的标准库经常会通过errno栈全局变量返回错误码，CGO中会添加一个额外返回值来支持：

```go
/* 
#include <errno.h>
static int div(int a, int b) {
	if(b == 0) {
		errno = EINVAL; 
		return 0;
	} 
	return a/b;
} 
*/ 
import "C" 
import "fmt"

func main() {
  v0, err0 := C.div(2, 1) 
  fmt.Println(v0, err0)
  v1, err1 := C.div(1, 0) 
  fmt.Println(v1, err1)
}
```

注意，即使C函数本身没有返回值（返回的是void），也需要使用两个返回值获取errno。

```go
//static void noreturn() {} 
import "C" import "fmt"

func main() {
  _, err := C.noreturn() 
  fmt.Println(err) 
}
```



#### C使用Go内存，copy方式

```go
package main

/* 
	void printString(const char* s) { printf("%s", s); } 
*/ 
import "C"

func printString(s string) {
  cs := C.CString(s) 
  defer C.free(unsafe.Pointer(cs))
  C.printString(cs)
}

func main() {
  s := "hello"
  printString(s)
}
```

`cs := C.CString(s)`这句话copy了Go内存，cs的内存是稳定的，所以需要手动释放。



#### C使用Go内存，高效版

在CGO调用的C语言函数返回前，cgo保证传入的Go语言内存在此期 间不会发生移动，C语言函数可以大胆地使用Go语言的内存！

```go
package main

/* 
#include<stdio.h>
void printString(const char* s, int n) { 
	int i;
	for(i = 0; i < n; i++) {
		putchar(s[i]);
	} 
	putchar('\n');
} 
*/ 
import "C"

func printString(s string) { 
  p := (*reflect.StringHeader)(unsafe.Pointer(&s)) 
  C.printString((*C.char)(unsafe.Pointer(p.Data)), C.int(len(s ))) 
}

func main() {
  s := "hello"
  printString(s)
}
```

不过需要小心的是在取得Go内存后需要马上传入C语言函数，不能保存到临时变量后再间接传入C语言函数。因为CGO只能保证在C函数调用之后被传入的Go语言内存不会发生移动，它并不能保证在传入C函数之前内存不发生变化。



#### C使用Go内存，长时持有

如果需要在C语言中访问Go语言内存对象，我们可以将Go语言内存对象在Go 语言空间映射为一个int类型的id，然后通过此id来间接访问和控制Go语言对象。

```go
package main
import "sync"

type ObjectId int32

var refs struct { 
  sync.Mutex 
  objs map[ObjectId]interface{} 
  next ObjectId 
}

func init() {
  refs.Lock() 
  defer refs.Unlock()
  
  refs.objs = make(map[ObjectId]interface{})
  refs.next = 1000
}

func NewObjectId(obj interface{}) ObjectId { 
  refs.Lock() 
  defer refs.Unlock()
	
  id := refs.next 
  refs.next++
	refs.objs[id] = obj
  return id
}

func (id ObjectId) IsNil() bool { 
  return id == 0 
}

func (id ObjectId) Get() interface{} { 
  refs.Lock()
  defer refs.Unlock()
  
	return refs.objs[id]
}

func (id *ObjectId) Free() interface{} {
  refs.Lock() 
  defer refs.Unlock()
  
  obj := refs.objs[*id] 
  delete(refs.objs, *id)
	*id = 0
  return obj
}
```

`ObjectId`是类似于句柄一样的东西。用完之后需要手工调用free方 法释放该对象ID。

使用示例：

```go
package main

/* 
extern char* NewGoString(char* ); 
extern void FreeGoString(char* ); 
extern void PrintGoString(char* );

static void printString(const char* s) { 
	char* gs = NewGoString(s); 
	PrintGoString(gs); 
	FreeGoString(gs); 
} */ 
import "C"

//export NewGoString 
func NewGoString(s *C.char) *C.char {
	gs := C.GoString(s)
	id := NewObjectId(gs)
  return (*C.char)(unsafe.Pointer(uintptr(id))) 
}

//export FreeGoString 
func FreeGoString(p *C.char) {
	id := ObjectId(uintptr(unsafe.Pointer(p)))	
  id.Free() 
}

//export PrintGoString 
func PrintGoString(s *C.char) {
  id := ObjectId(uintptr(unsafe.Pointer(p)))
  gs := id.Get().(string)
  print(gs) 
}

func main() { 
  C.printString("hello") 
}
```

这块没太看明白。 感觉这句是使用Go的内存`gs := id.Get().(string)`，然而是在Go环境中用的。。



#### 导出C函数不能返回Go内存

```go
package main
/*
extern int* getGoPtr();

static void Main() {
	int* p = getGoPtr();
	*p = 42;
}
*/
import "C"

func main() { C.Main() }

//export getGoPtr
func getGoPtr() *C.int {
	return new(C.int)
}
```

运行此代码会引起panic。其中getGoPtr返回的虽然是C语言类型的指针，但是内存本身是从Go语言的new函 数分配，也就是由Go语言运行时统一管理的内存。然后我们在C语言的Main函数中 调用了getGoPtr函数，此时默认将发送运行时异常。



#### Go调用C++

C++ 类定义

```c++
// my_buffer.h
#include <string>

struct MyBuffer {
    std::string* s_;

    MyBuffer(int size) { this->s_ = new std::string(size, char('\0'));}

    ~MyBuffer() { delete this->s_; }

    int Size() const { return this->s_->size(); }

    char* Data() { return (char*)this->s_->data(); }
};
```

定义C接口文件

```c
// my_buffer_capi.h

typedef struct MyBuffer_T MyBuffer_T;

MyBuffer_T* NewMyBuffer(int size);

void DeleteMyBuffer(MyBuffer_T* p);

char* MyBuffer_Data(MyBuffer_T* p);

int MyBuffer_Size(MyBuffer_T* p);
```

C接口的实现

```c
#include "./my_buffer.h"

extern "C" {
    #include "./my_buffer_capi.h"
}

struct MyBuffer_T: MyBuffer {
    MyBuffer_T(int size): MyBuffer(size) {}
    ~MyBuffer_T() {}
};

MyBuffer_T* NewMyBuffer(int size) {
    auto p = new MyBuffer_T(size);
    return p;
}

void DeleteMyBuffer(MyBuffer_T* p) {
    delete p;
}

char* MyBuffer_Data(MyBuffer_T* p) {
    return p->Data();
}

int MyBuffer_Size(MyBuffer_T* p) {
    return p->Size();
}
```

定义了MyBuffer_T集成自C++的MyBuffer。然后利用MyBuffer_T实现了接口定义的操作。

接下来是C接口函数转为Go函数。

```go
// my_buffer_capi.go
package main

/*
#cgo CXXFLAGS: -std=c++11
#include "my_buffer_capi.h"
*/
import "C"

type cgo_MyBuffer_T C.MyBuffer_T

func cgo_NewMyBuffer(size int) *cgo_MyBuffer_T {
	p := C.NewMyBuffer(C.int(size))
	return (*cgo_MyBuffer_T)(p)
}

func cgo_DeleteMyBuffer(p *cgo_MyBuffer_T) {
	C.DeleteMyBuffer((*C.MyBuffer_T)(p))
}

func cgo_MyBuffer_Data(p *cgo_MyBuffer_T) *C.char {
	return C.MyBuffer_Data((*C.MyBuffer_T)(p))
}

func cgo_MyBuffer_Size(p *cgo_MyBuffer_T) C.int {
	return C.MyBuffer_Size((*C.MyBuffer_T)(p))
}
```

到目前为止，还是有C类型出现的。下边将其封装为Go的对象。

```go
// my_buffer.go

package main

import "unsafe"

type MyBuffer struct {
	cptr *cgo_MyBuffer_T
}

func NewMyBuffer(size int) *MyBuffer {
	return &MyBuffer{
			cptr: cgo_NewMyBuffer(size),
		}
}

func (p *MyBuffer) Delete() {
	cgo_DeleteMyBuffer(p.cptr)
}

func (p *MyBuffer) Data() []byte {
	data := cgo_MyBuffer_Data(p.cptr)
	size := cgo_MyBuffer_Size(p.cptr)
	return ((*[1 << 31]byte)(unsafe.Pointer(data)))[0:int(size): int(size)]
}
```

使用

```go
package main

//#include <stdio.h>
import "C"

import "unsafe"

func main() {
	buf := NewMyBuffer(1024)
	defer buf.Delete()

	copy(buf.Data(), []byte("hello world\x00"))
	C.puts((*C.char)(unsafe.Pointer(&(buf.Data()[0]))))
}
```

