<a name="9d1f6ab7"></a>
### 1.1.1. Go编译

- 词法与语法分析 
   - 意义:解析源代码文件,将文件中字符串序列转换成Token序列
   - 把执行词法分析的程序称为词法解析器(lexer)
   - 语法解析的结果就是抽象语法树(AST)
   - 每个AST都对应一个单独的Go语言文件,这个抽象语法树中包括当前文件属于的包名,定义的常量,结构体和函数等
   - 如果发生错误,被语法解析器发现并将消息打印在标准输出上,编译过程直接中止
   - Go语言早期用lex做词法分析,后续还是使用Go语言实现词法分析器,自己写的词法分析器分析自己
- 类型检查和AST转换 
   - 编译器对语法树中定义和使用的类型进行检查
   - 遍历抽象节点树,保证当前节点上不会出现类型错误
   - 不仅对类型进行检查,还会对内置函数进行展开和改写,比如`make`关键字在这个阶段会根据子树的结构被替换成`makeslice` 或者 `makechan` 等函数
- 通用SSA(静态单赋值)生成 
   - 使用SSA特性,分析代码中无用变量和片段进行优化
   - 类型检查之后,就对Go语言项目全部函数进行编译,生成中间代码
   - 关键字和内置函数的功能其实是由语言的编译器和运行时共同完成的
- 机器代码生成 
   - 根据机器不同,生成不同的机器码

<a name="78e21b09"></a>
### 1.1.2. 函数

<a name="86fbce1f"></a>
#### 调用惯例

- C语言的函数的参数是通过寄存器和栈传递的 
   - x86_64的机器,6个(含6个)的参数会按照顺序分别使用 edi、esi、edx、ecx、r8d 和 r9d 六个寄存器传递，超过 6 个的剩余参数会通过栈进行传递
   - 函数的返回是通过eax 寄存器进行传递的,所以不支持多个返回值
- Go语言传递和接收参数使用的都是栈 
   - 但是需要注意,函数入参和出参的内存空间都需要调用方再栈上进行分配
   - 好处: 
      - 能够降低实现的复杂度(不需要考虑超过寄存器个数的参数应该如何传递)
      - 更方便的兼容不同的硬件(不同cpu的寄存器差别比较大)
      - 函数可以具有多个返回值(栈上的内存地址相比寄存器的个人是无限的,使用寄存器支持多个返回值也会非常困难,超出寄存器多个返回值也需要使用栈来传递)
   - 通过堆栈传递参数,入栈的顺序都是从右到左
   - 函数返回通过堆栈传递并由调用者预先分配内存空间

<a name="67441eb2"></a>
#### 参数的传递

- 无论是传递基本类型,结构体还是指针,都会对传递的参数进行拷贝
- 指针作为参数传入某一个函数的时候,其实在函数内部会对指针进行复制,也就是会同时出现两个指针指向原有的内存空间,所以Golang传指针也是传值
- 调用函数都是传值,接收方会对入参进行复制再计算

<a name="222ea65e"></a>
### 1.1.3. 接口

其本质就是引入一个中间层对不同的模块进行解耦,上层的模块就不需要依赖某一个具体的实现! Go语言中的接口`interface` 不仅是一组方法,还是一种内置的类型

Go语言中所有的接口的实现都是隐式的

<a name="27405e34"></a>
#### 接口类型

> interface{}类型并不表示任意类型, interface{}类型的变量再运行期间的类型只是interface{}
>  
> go/src/runtime/runtime2.go#144


<a name="67107751"></a>
##### 带有一组方法的接口

- 表示成 `iface`结构体

```go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}
Copy
```

<a name="cb6de3ca"></a>
##### itab结构体

```go
// layout of Itab known to compilers
// allocated in non-garbage-collected memory
// Needs to be in sync with
// ../cmd/compile/internal/gc/reflect.go:/^func.dumptypestructs.
type itab struct {
    inter *interfacetype
    _type *_type
    hash  uint32 // copy of _type.hash. Used for type switches.
    _     [4]byte
    fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
Copy
```

- `interfacetype` 是对 `_type`类型的简单封装
- `hash` 对 `_type.hash`的拷贝, 它会从 `interface` 到具体类型的切换时用于 快速判断目标类型和接口中类型是否一致
- `fun` 数组其实是一个动态大小的数组,如果数组中内容为空表示 `_type`没有实现`inter`接口,虽然这是一个大小固定的数组,但是在使用时会直接通过指针获取其中的数据并不会检查数组的边界,所以该数组中保存的元素数量是不确定的

<a name="be4bdcd2"></a>
##### 不带有任何方法的`interface{}`类型

- 表示成`eface`结构体
- 很常见,实现成一种特殊的类型

```go
type eface struct {
    _type *_type
    data  unsafe.Pointer
}
Copy
```

- `_type` 类型有点复杂.. 不看了..

<a name="95963c8a"></a>
#### 动态派发

运行期间选择具体的多态操作执行的过程,在Go语言中,对于一个接口类型的方法调用,会在运行期间决定具体调用该方法的哪个实现

动态派发会出现一些消耗,但一般项目中不可能只存在动态派发的调用,荔港南湾,如果开启默认的编译器优化,动态派发开销还会降低,所以对整体性能影响很小

结构体指针换成结构体,消耗区别有点大,原因是 Go 函数调用是值传递,会出现参数拷贝,所以对于大结构体,参数拷贝会消耗非常多资源,所以应该用指针来传递大结构体

<a name="a36d1e6c"></a>
### 1.1.4. 数组与切片

<a name="0e67d4b0"></a>
#### 数组

- 数组是由相同类型元素的集合组成的数据结构
- 数组大小在初始化之后就无法改变
- 编译期间,数组类型`Array`包含两个结构,一个是元素类型`Elem`, 另一个是数组的大小上限`Bound`,这两字段组成了数组类型
- 数组创建可以显式指定数组的大小,也可以根据源代码自定推断数组的大小,不过后者在编译期间会被转换前一种 -

<a name="08a74860"></a>
##### 数组上限推导

- 显式创建,变量类型会在编译进行到 类型检查 阶段就会被推断出来
- 非显式创建,会在 编译进行 类型检查 时候也会创建一个 `Array` 类型,不过`Bound = -1` 后面会推到该数组大小
- 所以`[...]T`类型的声明不是在运行是被推导的,会在类型检查期间就被推断出正确的数组大小

<a name="58aebb1d"></a>
##### 语句转换

- `[...]T{1, 2, 3}`与`[3]T{1, 2, 3}` 运行的时候 是等价的,理由如上
- 根据数组元素不同,会有不同的优化 
   - 当元素数量小于或者等于4个时,会直接将数组中的元素放置在栈上
   - 当元素数量大于4个时,会将数组中的元素放置到静态区并在运行时取出
- 总结: 数组元素个数小于等于4个,所有变量会直接在栈上初始化,如果数组数量大于4个,变量就会在静态存储区初始化然后拷贝到栈上,转换后的代码才会进入 中间代码生成 和机器码生成阶段,最后生成可执行的二进制文件

<a name="97e3ab54"></a>
##### 访问和赋值

- 数组在内存中其实就是一连串的内存空间,表示数组的方法就是一个指向数组开头的指针
- 数组访问越界的判断也是在编译期间由静态类型检查完成的
- 数组的赋值和更新 也会在生成SSA期间就计算出数组当前元素的内存地址,然后修改当前内存地址的内容
- 数组的寻址或者赋值都是编译阶段完成,没有运行时的参与

<a name="261b8a9f"></a>
#### 切片

- 切片其实就是动态数组,长度不固定
- 切片在编译期间应该只会包含切片中的元素类型 `Slice{Elem: elem}`
- 切片的操作基本都是在编译期间完成的,除了访问切片的长度,容量或者其中的元素之外,使用`range`遍历切片时也是在编译期间被转换成了形式更简单的代码

<a name="d9a4564e"></a>
##### 切片结构

```go
type SliceHeader struct {
    Data uintptr // 指向数组的指针
    Len  int // 当前切片长度
    Cap  int // 切片的容量
}
Copy
```

<a name="70749714"></a>
##### 切片初始化

- 字面量 
   - `slice := []int{1, 2, 3}` -
- 关键字 
   - slice := make([]int, 10)

<a name="97773362"></a>
##### 切片追加

-  追加`append` 会根据`inplace` 在中间代码生成阶段转换不同流程,一种是追加之后,不需要赋值回原有的变量`append(slice, 1, 2, 3)`,一种是需要赋值给原有的变量`slice = apennd(slice, 1, 2, 3)` 
-  先对切片结构体进行解构获取数组指针,大小和容量,如果新切片大小大于容量,那么会使用 
```go
growslice
```
<br />对切片进行扩容并将新的元素依次加入切片并创建新的切片 

   -  如果期望容量大于当前容量的两倍就会使用期望容量 
   -  如果当前切片容连小于1024就会将容连翻倍 
   -  如果当前切片容量大于1024就会每次增加25%的容量,知道新容量大于期望容量 
```go
func growslice(et *_type, old slice, cap int) slice {
newcap := old.cap
doublecap := newcap + newcap
if cap > doublecap {
  newcap = cap
} else {
  if old.len < 1024 {
      newcap = doublecap
  } else {
      for 0 < newcap && newcap < cap {
          newcap += newcap / 4
      }
      if newcap <= 0 {
          newcap = cap
      }
  }
}
Copy
```
 

-  确定了切片的容量,就可以算内存占用了, 内存占用= 目标容量* 元素大小 

<a name="5f0ab363"></a>
##### 切片拷贝

- `copy(a, b)` 会在编译期间转换成`slicecopy`函数
- 切片中全部元素通过`memmove`或者数组指针的方式将整块内存中的内容拷贝到目标的内存区域,所以大切片拷贝需要注意性能影响,不过比一个个的复制要有更好的性能

<a name="18046da4"></a>
##### 数组和切片总结

- 数组大多都是在 编译期间都转换成内存的直接读写
- 切片很多功能都需要运行时的支持

<a name="c9b0039f"></a>
### 1.1.5. 哈希表

-  哈希表示键值之间隐射关系 
-  哈希函数 
   - 理想状态是 不同的键映射到不同的唯一索引上,要求哈希函数的输出范围大于输入范围,但是实际使用时这种理想状态是不可能实现的

<a name="b0a4f985"></a>
#### 哈希冲突解决

- 开放寻址法 
   - 在一堆数组对元素进行探测和比较以判断待查找的目标键是否存在当前的哈希表中
   - 初始化哈希表时会创建一个新的数组,如果哈希表写入新的数据发生了冲突,就会将键值对写入到下一个不为空的位置
   - 查找时,按照我的理解,应该是先哈希key,找到对应的位置上,然后对比key,如果不一样,继续往下找,除非找到对应的key或者内存为空为止
   - 数组中元素数量与数组大小的比值叫做装载因子,随着装载因子增加,线性探索的评价用时就会逐渐增加,到百分之七十之后性能明显下降,一旦到百分之百,整个哈希表就会完全失效
- 拉链法 
   - 实现比较简单,存储节点的内存是动态申请的,比较节省空间
   - 就是数组加上链表组合起来实现哈希表,数组中每个元素都是一个链表
   - 插入时,先对key进行hash,找到数组上对应的点(桶),然后遍历桶里的链表,如果有相同的就修改,没有就加在链表最末尾
   - 性能比较好的哈希表中,每个桶里大约有0或者1个元素,偶尔会有2到3个,很少会超过这个数

<a name="8a95a6d1"></a>
#### 结构

```go
type hmap struct {
    count     int // 用于记录当前哈希表元素数量,这个元素让我们不在需要去遍历整个哈希表来获取长度
    flags     uint8
    B         uint8  // 表示当前哈希表持有的 `buckets` 数量,因为哈希表扩容是以2倍进行的,所以这里会使用对数来存储, 简单理解成 `len(buckets) == 2^B` 
    noverflow uint16
    hash0     uint32 // 哈希的种子,这个值会在调用哈希函数时作为参数传入进去,主要作用是为哈希函数的结果引入一定的随机性

    buckets    unsafe.Pointer
    oldbuckets unsafe.Pointer // 哈希在扩容时用于保存之前的 `buckets`的字段,它的大小是当前`buckets`的一半
    nevacuate  uintptr

    extra *mapextra
}
Copy
```

<a name="a53afead"></a>
#### 字面量

-  编译时用`maplit`进行初始化 
-  当哈希表中元素数量小于等于25个时,编译器会调用`addMapEntries`将结构体转成单独的键值对,比如 hash["a"] = 1, hash["b"] = 2 
-  当元素数量超过25个,会在编译期间创建两个数组分别存储键和值的信息,这些键值会通过一个for循环假如目标的哈希 
```go
hash := make(map[string]int, 26)
vstatk := []string{"1", "2", "3", ... ， "26"}
vstatv := []int{1, 2, 3, ... , 26}
for i := 0; i < len(vstak); i++ {
  hash[vstatk[i]] = vstatv[i]
}
Copy
```
 

<a name="9d996717"></a>
#### 运行时

- 编译期间`make`转换成`makemap`来创建哈希表
- 根据`B`算出需要创建的桶数量,在内存里分配一片连续空间用于存储数据,创建过程中,还会创建一些用于保存溢出数据的桶,数量 2^(B-4)个
- 哈希表的桶最多只能存储8个元素,如果超过8个,效率会下降,所以会进行扩容或者使用额外的桶存储溢出的数据,不会让桶里数据超过8个

<a name="d5c0ebde"></a>
#### 访问

-  类型检查阶段,类似`hash[key]`的`OINDEX`操作都会被转换成`OINDEXMAP`操作,中间代码生成阶段,在`walkexpr`中将这些`OINDEXMAP`转成 
```go
v     := hash[key] // => v     := *mapaccess1(maptype, hash, &key)
v, ok := hash[key] // => v, ok := mapaccess2(maptype, hash, &key)
Copy
```
 

-  当接收参数就一个时,使用`mapaccess1`函数,如果多加一个是否存在的布尔值就会使用`mapaccess2` 
-  在这个函数中我们首先会通过哈希表设置的哈希函数、种子获取当前键对应的哈希，再通过 bucketMask 和 add 函数拿到该键值对所在的桶和哈希最上面的 8 位数字，这 8 位数字最终就会与桶中存储的 tophash 作对比，每一个桶其实都存储了 8 个 tophash，就是编译期间的 topbits 字段，每一次都会与桶中全部的 8 个 uint8 进行比较，这 8 位的 tophash 其实就像是一级缓存，它存储的是哈希最高的 8 位，而选择桶时使用了桶掩码使用的是最低的几位，这种方式能够帮助我们快速判断当前的键值对是否存在并且减少碰撞,每一个桶都是一整片的内存空间，当我们发现某一个 topbits 与传入键的 tophash 匹配时，通过指针和偏移量获取哈希中存储的键并对两者进行比较，如果相同就会通过相同的方法获取目标值的指针并返回。另一个同样用于访问哈希表中数据的 mapaccess2 函数其实只是在 mapaccess1 的基础上同时返回了一个标识当前数据是否存在的布尔值 

<a name="358e76ae"></a>
#### 写入

- 当哈希表没有处于扩容状态并且装载因子超过了6.5或者存在了太多溢出的桶,调用`hashGrow`对当前哈希表进行扩容
- 装载因子是同时由 `loadFactorNum` 和 `loadFactDen` 两个参数决定的，前者在 Go 源代码中的定义是 13 后者是 2，所以装载因子就是 6.5
- 如果桶满了,会调用 `newoverflow` 创建一个新的桶或者使用`hmap`预先在`noverflow`中创建好的桶来保存数据,新创建的桶的指针会被追加到已有桶中,与此同时,溢出桶的创建会增加哈希表的`noverflow`计数器
- 如果哈希表存储的键值是指针类型,其实就会被当前的键值对分别申请一块新的内存空间,并在插入的位置通过`eypedmemmove`将键移动到申请的内存空间,最后返回键对应的地址

<a name="52228f8a"></a>
#### 扩容

- 如果扩容是溢出的桶太多,那么就是 `sameSizeGrow`
- 如果是一次不改变大小的扩容,`evacDst`结构体只会初始化一个,当哈希表容量翻倍时,一个桶中的元素会被分流到新创建的两个桶中,这两个桶会被`evacDst`数组引用

<a name="2f4aaddd"></a>
#### 删除

- `delete`关键字,将某一个键对应的元素从哈希表中删除,无论该键对应的值是否存在,这个内建的函数都不会返回任何的结果
- 如果在删除期间遇到哈希表的扩容,就会对即将操作的桶进行分流,随后找到桶中的目标元素并根据数据的类型调用`memclrHasPointers` 或者 `memclrNoHeapPointers` 函数完成键值对的删除
- `delete`在类型检查阶段被转换成`ODELETE`操作,然后在 `SSA中间代码生成`时被转换成`mapdelete`函数簇

<a name="01df1f6f"></a>
#### 哈希表总结

- Go语言用拉链法来解决哈希碰撞
- 哈希在每一个桶中存储键对应哈希的前8位,当对哈希进行操作时,这些 `tophash`就成了一级缓存帮助哈希快速遍历桶中元素
- 每个桶只能存储8个键值对,一旦某个桶超过8个,新的键值对会被存储到哈希的溢出桶中
- 当键值对数量的增加,装载因子升高,到一定范围后,会出发扩容操作,扩容时将桶的数量分配,元素再分配的过程也是在调用写操作时增量进行的,不会造成性能的瞬时巨大波动

<a name="8222bbbe"></a>
### 1.1.6. 字符串

- Go语言中的字符串是一个只读的字节数组切片
- 如果代码中存在的字符串,会被编译期间标记成只读数据`SRODATA`,但这只是表示这个字符串会被分配到只读的内存空间并且这段内存不会被修改,但是运行时,依然可以将这段内存拷贝到其他的堆或者栈上,同时将变量的类型修改成 `[]byte`,在修改之后通过类型转换变成`string`,如果想直接修改`string`类型变量的内存空间,是不支持的!

<a name="8a95a6d1-1"></a>
#### 结构

- 在运行时,用`StringHeader`结构体进行表示, 在运行时包的内部其实有一个私有的结构`stringHeader`, 它有着相同的结构,只是用于存储数据的`Data`字段使用了`unsafe.Pointer`类型

```go
type StringHeader struct {
    Data uintptr
    Len  int
}
Copy
```

<a name="aa357cb9"></a>
##### 解析

-  解析器在 词法分析 时就完成的 
-  两种字面量的方式可以声明字符串,一种是 双引号,一种是 使用反引号 
   -  双引号用于简单的 单行的字符串,内部出现双引号需要用 
```go
\
```
<br />符号避免编译器的解析错误 

      -  标准字符串使用双引号表示开头和结尾 
      -  标准字符串中需要使用反斜线`\`来`escape`双引号 
      -  标准字符串中不能出现隐式的换行符号 
```go
\n
```
  

   -  反引号 可以摆脱单行的限制,内部可以直接使用 
```go
"
```
<br />,写 <br />或者其他数据时候非常方便 

      - 只需要把非反引号的所有字符都划分到当前字符串的范围中

<a name="a6b363bb"></a>
##### 拼接

- 使用`+`符号,编译器在检查阶段将`OADD`节点转换成`OADDSTR`类型的节点,然后在SSA中间代码生成的阶段调用`addstr`函数
- `addstr`函数在编译期间合选择合适的函数对字符串进行拼接,如果拼接字符串小于或者等于5个,那么活直接调用`concatstring{2,3,4,5}` 等一系列函数,如果超过5个就会直接选择`concatstrings`传入一个数组切片
- 不管选择哪个函数,最终都调用`concatstrings`,这个函数会先对传入的切片参数进行遍历,首先会过滤空字符串比国内获取拼接后的字符串长度

<a name="f3c723ec"></a>
##### 类型转换

-  `string(bytes)`会在编译期间转换成`slicebytetostring`的函数调用 
-  字符串到字节数组转换使用 
```go
stringtoslicebyte
```
 

   - 使用传入的缓冲区或者根据字符串的长度调用`rawbyteslice`创建一个新的字节切片,`copy`关键字会将字符串中的内容拷贝到新的字节数组中

<a name="60b4beb1"></a>
##### 字符串总结

- 字符串内容是只读,切片是可读写的,所以相互转换都是对内容进行拷贝,拷贝性能会因为字符串数组和字节长度的增长而增长,所以需要转换需要注意性能问题

<a name="3a79deae"></a>
### 1.1.7. for 和 range

-  `for` 和 `for...range` 经过优化后,变成一样 
-  结果是???? 

```go
func main() {
    arr := []int{1, 2, 3}
    for _, v := range arr {
        arr = append(arr, v)
    }
    fmt.Println(arr)
}
Copy
```

- 只遍历了切片里的元素,追加并不会导致循环次数增加,然后最后停了下来

```go
func main() {
    arr := []int{1, 2, 3}
    newArr := []*int{}
    for _, v := range arr {
        newArr = append(newArr, &v)
    }
    for _, v := range newArr {
        fmt.Println(*v)
    }
}
Copy
```

-  结果是 , 如何避免 
-  for 经典循环 
-  for...range 范围循环 编译器会在编译期间将带有`range`的循环变成普通的经典循环,这个过程发生在 SSA中间代码 阶段,所有的`range`都会被`walkrange` 函数转换成只包含基本表达式的语句,不包含任何复杂的结构 

<a name="75dffc40"></a>
##### 遍历数组和切片

- 对于所以的`range`循环,Go语言都会在编译期间将原切片或者数组赋值给一个新的变量`ha`,在赋值的过程中其实就发生了拷贝,所以我们遍历的切片其实已经不是原有的切片变量了!
- 当同时遍历索引和元素的`range`循环时,Go语言会额外创建一个新的`v2`变量存储切片中的元素,循环中使用的这个变量v2会在每一次迭代中都被重新赋值,在赋值时也发生了拷贝, 所以我们想要访问数组中元素所在的地址,不应该直接获取`range`返回的`v2`变量的地址`&v2`,想要解决这个问题应该使用`&a[index]`这种方式获取数组中元素对应的地址

<a name="063c4306"></a>
##### 遍历哈希

- 哈希表遍历会随机(`fastrand`函数)选择开始的位置,然后依次遍历桶中的元素,桶中元素如果被遍历完,就会遍历当前桶对应的溢出桶,溢出桶都遍历结束之后才会遍历哈希中下一个桶,直到所有的桶都被遍历完

<a name="6ab3f947"></a>
##### 遍历字符串

- 遍历过程中会获取字符串中索引对应的字节,然后将字节转换成`rune`,我们在遍历字符串时拿到的值都是`rune`类型的变量

<a name="c3050656"></a>
##### 遍历通道

- 循环会使用`<-ch`从管道中取出等待处理的值,这个操作会调用`chanrecv2`并阻塞当前的协程,当`chanrecv2`返回时会根据`hb`来判断当前的值是否存在,如果不存在就意味着当前的管道已经被关闭了,在正常情况下都会为`v1`赋值并清除`hv1`中的数据,然后会陷入下一次的阻塞等待接受新的数据

<a name="7117a162"></a>
### 1.1.8. defer

- 作用域结束之后执行函数的关键字
- `defer`实现是由编译器和运行时共同完成的

```go
func main() {
    {
        defer fmt.Println("defer runs")
        fmt.Println("block ends")
    }

    fmt.Println("main ends")
}
Copy
```

- 不是在当前代码块的作用域时执行的, `defer`只会在当前函数和方法返回之前被调用

```go
type Test struct {
    value int
}

func (t Test) print() {
    println(t.value)
}

func main() {
    test := Test{}
    defer test.print()
    test.value += 1
}
Copy
```

稍微改动一下

```go
type Test struct {
    value int
}

func (t *Test) print() {
    println(t.value)
}

func main() {
    test := Test{}
    defer test.print()
    test.value += 1
}
Copy
```

- 还是进行的值传递,不过发生复制的是指向`test`的指针,上面那个复制的是结构体,这段是复制的指针,修改`test.value`时,`defer`捕获的指针其实就能够访问到修改后的变量了

<a name="90ed305f"></a>
##### 实现原理

```go
type _defer struct {
    siz     int32
    started bool
    sp      uintptr
    pc      uintptr
    fn      *funcval
    _panic  *_panic
    link    *_defer
}
Copy
```

- `sp`和`pc`分别指向了栈指针和调用方的程序计数器,`fn`存储的就是向`defer`关键字中传入的函数
- `defer` 关键字在编译期间的SSA阶段才被`stmt`函数处理的,中间详情不表
- 运行时,每一个`defer`关键字都会被转换成`deferproc`,这个函数里会为`defer`创建一个新的`_defer`结构体并设置它的`fn`,`pc`和`sp`参数,并将`defer`相关的函数都拷贝到紧挨着结构体的内存空间中

<a name="25f9c7fa"></a>
##### 总结

- `defer`关键字会在编译阶段被转换成`deferproc`的函数并在函数返回之前插入`deferreturn`指令,在运行期间,每一次`deferproc`的调用都会将一个新的`_defer`结构体追加到当前Goroutine持有的链表头,而`deferreturn`会从Goroutine中取出`_defer`结构并以此执行,所有的`_defer`结构执行成功之后当前函数才返回!

<a name="4b89bdf3"></a>
### 1.1.9. panic 和 recover

- panic 能改变程序的控制流,当函数调用执行panic,它会立刻停止执行函数其他的代码,而是会运行其中的`defer`函数,执行成功返回到调用方
- 调用导致panic和直接调用`panic`类似,执行所有的`defer`函数并返回到它的调用方,这个过程会一直进行到当前的Goroutine的调用栈不包含任何的函数,这时整个程序才会崩溃
- `panic`导致的`恐慌`状态其实可以被`defer`中的`recover`中止,`recover`是一个只在`defer`中能够发挥作用的函数,在正常的控制流程中,`recover`会直接返回`nil`并没有任何的作用,如果当前的Goroutine发生了`恐慌`,`recover`就能够捕获到`panic`抛出的错误并阻止`恐慌`的继续传播

<a name="0fea7c47"></a>
##### 数据结构

```go
type _panic struct {
    argp      unsafe.Pointer // 指向`defer`调用时参数的指针
    arg       interface{} // 调用`panic`时传入的参数
    link      *_panic // 指向更早调用的`_panic`结构
    recovered bool // 当前的`_panic`是否被`recover` 恢复
    aborted   bool // 表示当前的`panic`是否被强行终止
}
Copy
```

1. 获取当前`panic`调用所在的Goroutine协程
2. 创建并初始化一个 `_panic`结构体
3. 从当前Goroutine的链表获取一个`_defer`结构体
4. 如果当前的`_defer`存在,调用`reflectcall`执行`_defer`中的代码
5. 将下一位的`_defer`结构设置到Goroutine上并返回到3
6. 调用`fatalpanic`中止整个程序(会在中止整个程序之前可能会通过`printpanics`打印出全部的`panic`消息以及调用时传入的参数)

<a name="897a4a74"></a>
##### panic和recover总结

-  编译过程中会将`panic`和`recover` 分别转换成`gopanic` 和`gorecover`函数,同时将`defer`转换成`deferproc` 函数并在调用`defer`的函数和方法末尾增加`deferreturn`的指令 
-  在运行过程中遇到`gopanic`方法时,会从当前Goroutine中取出`_defer`的链表并通过`reflectcall`调用用于收尾的函数 
-  如果在 
```go
reflectcall
```
<br />调用时遇到了 <br />就会直接将当前的 <br />标记成 <br />并返回 <br />传入的参数(在这时 <br />就能够获取到 <br />的信息) 

   - 在这次调用结束后,`gopanic`会从`_defer`结构体中取出程序计数器`pc`和栈指针`sp`并调用`recovery`方法进行恢复
   - `recovery`会根据传入的`pc`和`sp`跳转到`deferproc`函数
   - 编译器自动生成的代码会发现`deferproc`的返回值不为`0`,这时就会直接跳到`deferreturn`函数中并恢复到正常的控制流程(依次执行剩余的`defer`并正常退出)
-  如果没有遇到`gorecover`就会一次遍历所有的`_defer`结构,并在最后调用`fatalpanic`中止程序,打印`panic`参数并返回错误码`2` 

<a name="c3d44a1c"></a>
### 1.1.10. make 和 new

-  
```go
make
```
<br />用于创建 切片,哈希表和管道等内置数据结构 

   - 在 类型检查 阶段,根据不同类型转换成`OMAKESLICE`,`OMAKEMAP`,`OMAKECHAN`三种不同类型的节点
-  
```go
new
```
<br />用于分配并创建一个指向对应类型的指针 

   - 在 `SSA 代码生成` 阶段经过 `callnew` 函数的处理,如果请求创建的类型大小是0,那么就会返回一个表示空指针的`zerobase`变量,在遇到其他情况会将关键字转换成`newobject`
   - 如果声明的变量或者参数不需要在当前作用域外`生存`,不会初始化在当前函数的栈中并随着函数调用的结束而被销毁

<a name="eefd0d6f"></a>
### 1.1.11. 上下文 Context

- Go语言中每一个请求都是通过一个单独的Goroutine进行处理的,`Context`主要作用就是在 不同的 Goroutine 之间同步请求特定的数据,取消信号以及处理请求的截至日期
- 每一个 `Context`都会从最顶层的 Goroutine 一层一层传递到最下层,如果没有 `Context`,上层执行操作出现错误时,下层不会收到错误而会继续执行下去
- 减少额外的资源的消耗,还能携带以请求为作用域的键值对信息

<a name="54ea89b4"></a>
##### 接口

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
Copy
```

-  `Deadline` 方法需要返回当前`Context` 被取消的时间,也就是完成工作的截至日期 
-  `Done` 方法需要返回一个 Channel, 这个 Channel 会在当前工作完成或者上下文被取消之后关闭, 多次调用 `Done` 方法会 返回同一个 Channel 
-  
```go
Err
```
<br />方法会返回当前 <br />结束的原因, 它只会在 <br />返回的 Channel 被关闭才会返回非空的值 

   - 如果当前 `Context` 被取消就会返回 `Canceled` 错误
   - 如果当前 `Context` 超时就会返回 `DeadlineExceeded` 错误
-  `Value` 方法会从 `Context` 中返回键对应的值,对于同一个上下文来说,多次调用`Value` 并传入相同的`Key` 会返回相同的结果,这个功能可以用来传递特定的数据 
-  `Backgroud` 和 `TODO` 方法在某种层面上看也是互为别名,两者没有太大区别,不过`context.Background()` 是上下文中最顶层的默认值,所有其他的上下文都应该从`context.Background()`演化出来 
-  我们应该只在不确定时使用`context.TODO()`,在多数情况下如果函数没有上下文作为入参,我们往往都会使用`context.Background()` 作为起始的`Context`向下传递 
-  `WithCancel` 方法能从 `Context` 中创建出一个新的子上下文,同时还会返回用于取消该上下文的函数,也就是`CancelFunc` 
-  `WithDeadline` 和 `WithTimeout` 也都能创建可以被取消的上下文,`WithTimeout` 只是`context` 包为我们提供的便利方法,能让我们更方便地创建`timeCtx` 

<a name="97fbfb9b"></a>
## 1.2. 定时器

```go
type timer struct {
    tb *timersBucket
    i  int

    when   int64
    period int64
    f      func(interface{}, uintptr)
    arg    interface{}
    seq    uintptr
}
Copy
```

-  `timer` 就是 Golang 定时器内部表示,每一个`timer` 其实都存在堆中 
-  `tb` 就是用于存储当前定时器的桶 
-  `i` 是当前定时器在堆中的索引,可以通过这两个变量找到当前定时器在堆中的位置 
-  `when` 表示当前定时器(Timer) 被唤醒的时间 
-  `period` 表示两次被唤醒的间隔,每当定时器被唤醒时都会调用`f(args,now)` 函数并传入`args` 和当前时间作为参数 
-  这里的`timer`作为一个私有结构体其实只是定时器的运行时表示,`time` 包对外暴露的定时器是如下结构 
```go
type Timer struct {
  C <-chan Time
  r runtimeTimer
}
Copy
```
 

-  `Timer` 定时器必须通过`NewTimer` 或者 `AfterFunc` 函数进行创建,其中的`runtimeTimer` 其实就是上面的`timer`结构体, 当定时器失效时,失效的时间就会被发送给当前定时器持有的Channel`C`, 订阅管道中消息的Goroutine就会接收到当前定时器失效的时间 

<a name="d9ac9228"></a>
##### 创建

-  
```go
time
```
<br />包对外提供了两种创建定时器的方法 

   -  `NewTimer` 接口创建用于通知触发时间的Channel,调用 `startTimer` 方法并返回一个创建指向`Timer`结构体的指针 
   -  `AfterFunc` 也提供了相似的结构,与上面不同的是,它只会在定时器到期时调用传入的方法 
   -  
```go
startTimer
```
<br />是创建定时器的入口,所有的定时器的创建和重启基本上都需要这个函数 

      - 调用`addTimer`函数,首先通过`assignBucket`方法为当前定时器选择一个`timersBucket` 桶,根据当前的Goroutine所在处理器P的id选择一个合适的桶,随后调用`addTimerLocked`方法将当前定时器加入桶中
      - `addtimerLocked` 会先将最新加入的定时器加到队列的末尾,随后调用`siftipTimer`将当前定时器与四叉树(或者四叉堆)中的父节点进行比较,保证父节点的到期时间一定小于子节点

<a name="d6c6e029"></a>
##### 触发

- 定时器的触发都是由`timerproc`中的一个双层`for`循环控制的,外层的`for`循环主要负责对当前的`Goroutine` 进行控制,它不仅会负责锁的获取和释放,还会在合适的时机触发当前Goroutine的休眠
- 内部循环 
   - 如果桶中不包含任何定时器就会直接返回并陷入休眠等待定时器加入当前桶
   - 如果四叉树最上面的定时器还没有到期会通过`notetsleepg`方法陷入休眠等待最近定时器的到期
   - 如果四叉树最上面的定时器已经到期 
      - 当定时器 `preiod > 0` 就会设置下一次会触发定时器的时间并将当前定时器向下移动到对应位置
      - 当定时器`preios <= 0` 就会将当前定时器从四叉树中移除
   - 在每次循环的最后都会从定时器中取出定时器中的函数,参数 和序列号并调用函数触发该计数器
- 使用`NewTimer`创建的定时器,传入的函数时`sendTime`,它会将当前时间发送到定时器持有的Channel中,而使用`AfterFunc` 创建的定时器,在内层循环中调用的函数就会是调用方法传入的函数了

<a name="e621d713"></a>
##### 休眠

- `timeSleep` 会创建一个新的`timer`结构体,在初始化的过程中我们会传入当前Goroutine 应该被唤醒的时间以及唤醒时需要调用的函数`goroutineReady`,随后会调用 `goparklock` 将当前GOroutine陷入休眠状态,当定时器到期时也会调用 `goroutineReady` 方法唤醒当前的Goroutine
- `time.Sleep` 方法其实只是创建了一个会在到期时唤醒当前Goroutine的定时器并通过`goparkunlock`将当前的协程陷入休眠状态等待定时器触发的唤醒

<a name="Ticker"></a>
##### Ticker

- 除了只用于一次的定时器(Timer)之外,Go语言的`time`包中还提供了用于多次通知的`Ticker`计时器,计时器中包含了一个用于接受通知的Channel 和一个定时器,这个两个字段组成了用于连续多次触发事件的计时器
- 想要在Go中创建一个计时器只有两种方法,一种是使用`NewTicker`方法显示的创建`Ticker`计时器指针,另一种可以直接通过`Tick`方法获取一个会定期发送消息的Channel
- 每一个`NewTicker`方法开启的计时器都需要在不需要使用时调用`Stop`进行关闭,如果不显示调用`Stop`方法,创建的计时器就没有办法被垃圾回收,而通过`Tick`创建的计时器由于只对外提供了Channel,所以是一定没有办法关闭的,我们一定要谨慎使用这一接口创建计时器

<a name="ebad1dc2"></a>
##### 性能分析

- 定时器在内部使用四叉树的方式进行实现和存储,高并发的场景下会有比较明显的性能问题

<a name="13401fb2"></a>
## 1.3. 同步原语与锁

- 锁的主要作用就是保证多个线程或者Goroutine在访问同一片内存时不会出现混乱的问题
- 锁其实是一种并发编程中的同步原语(Synchronization Promitives)

<a name="ada44b25"></a>
#### 基本原语

- Go语言在 `sync` 包中提供了用于同步的一些基本原语,包括常见的互斥锁`Mutex` 与读写互斥锁 `RWMutex`以及 `Once`, `WaitGroup`

<a name="Mutex"></a>
##### Mutex

-  Go语言中的互斥锁在`sync`中, 由`state` 和`sema` 组成, `state` 表示 当前互斥锁的状态, 而`sema` 真正用于控制锁状态的信号量, 这两个加起来只占8字节空间的结构体就表示了Go语言中的互斥锁 
```go
type Mutex struct {
  state int32
  sema  uint32
}
Copy
```
 

-  互斥锁的状态是用`int32`来表示的,但是锁的状态并不是互斥的,它的最低三位分别表示`mutexLocked`,`mutexWoken`,`mutexStarving`,剩下的位置都用来表示当前有多少个Goroutine等待互斥锁被释放 
-  互斥锁在被创建出来时,所有的状态位的默认值都是`0`,当互斥锁被锁定时,`mutexLocked` 就会被置成`1`,当互斥锁被在正常模式下被唤醒`mutexWoken`就会被置成`1`,`mutexStarving`用于表示当前的互斥锁进入了状态,最后的几位是在当前互斥锁上等待的Goroutine个数 

<a name="252bba5a"></a>
##### 饥饿模式 ✦ ✦ ✦ ✦

- 饥饿模式是在Go语音`1.9`版本引入的特性,主要功能就是保证互斥锁的获取的`公平性`
- 互斥锁可以同时处于两种不同的模式,也就是正常模式和饥饿模式, 在正常模式下,所有锁的等待者都会按照先进先出的顺序获得锁,但是如果一个刚刚被唤醒的 Goroutine 遇到了新的Goroutine 进程也调用了 Lock 方法时,大概率获取不到锁,为了减少这种情况的出现,防止Goroutine被 饿死, 一旦 Goroutine超过1ms 没有获取到锁,它就会将当前互斥锁切换饥饿模式
- 在饥饿模式中,互斥锁会被直接交给等待队列最前面的 Goroutine,新的Goroutine在这时不能获取锁,也不会进入自旋的状态,它们只会在队列的末尾等待,如果一个Goroutine获得了互斥锁并且它是队列中最末尾的协程或者它等待的时间少于1ms,那么当前的互斥锁就会被切换回正常模式
- 相比于饥饿模式,正常模式下的互斥锁能够提供更好的性能,饥饿模式的主要作用就是避免一些Goroutine由于陷入等待无法获取锁而造成较高的尾延迟,这也是对`Mutex`的一个优化

<a name="bba1928e"></a>
##### 加锁

> 饥饿模式不会进入进入自旋,那么如果是正常模式转成饥饿模式,自旋还有么??????


-  互斥锁`Mutex`的加锁是靠`Lock` 方法完成的,最新的 Go语言源代码中已经将`Lock` 方法进行了简化,方法的主干只保留了最常见,简单并且快速的情况,当锁的状态是 `0` 时直接将`mutexLocked` 位置成 `1` 
-  当`Lock` 方法被调用时`Mutex`的状态不是 `0` 时就会进入 `lockSlow` 方法尝试通过自旋或者其他方法等待锁的释放并获取互斥锁 
-  自旋其实是在多线程同步的过程中使用的一种机制,当前的进程在进入自旋的过程中会一直保持CPU的占用,持续检查某个条件是否为真,在多核的CPU上,自旋的优点是避免了Goroutine的切换,如果使用恰当会对性能带来非常大的增益 
-  在 Go语言 的 
```go
Mutex
```
<br />互斥锁中,只有在普通模式下才可能进入自旋,除了模式的限制之外, <br />方法中会判断当前方法是否可以进入自旋,进入自旋的条件非常苛刻 

   - 运行在多CPU的机器上
   - 当前Goroutine 为了获取该锁进入自旋的次数小于四次
   - 当前机器上至少存在一个正在运行的处理器`P`并且处理的运行队列是空的
-  `runtime_SemacquireMutex` 方法主要作用就是通过`Mutex`的使用互斥锁中的信号量保证资源不会被两个Goroutine获取,从这里我们就能看出`Mutex`其实就是对更底层的信号量进行封装,对外提供更加易用的API,`runtime_SemacquireMutex` 会在方法中不断调用 `goparkunlock`将当前 Goroutine陷入休眠等待信号量可以被获取, 
-  一旦当前Goroutine 可以获取信号量,就证明互斥锁已经被解锁,该方法就会立刻返回,`Lock`方法的剩余代码也会继续执行下去了,当前互斥锁处于饥饿模式时,如果该Goroutine是队列中最后的一个Goroutine 或者等待锁的时间小于 `starvationThresho1dNs(1ms)` 当前Goroutine 就会直接获得互斥锁并且从饥饿模式中退出并获得锁 

<a name="fa7ffa2d"></a>
##### 解锁

- 互斥锁的解锁过程相比之下就非常简单,`Unlock` 方法会直接使用`atomic` 包提供的`AddInt32`,如果返回的新状态不等于 `0` 就会进入 `unlockSlow` 方法
- `unlockSlow` 方法首先会对锁的状态进行校验,如果当前互斥锁已经被解锁过了就会直接抛出异常`sync: unlock of unlocked mutex` 中止当前程序,在正常情况下会根据当前互斥锁的状态是正常模式还是饥饿模式进入不同的分支!
- 如果当前互斥锁的状态是饥饿模式就会直接调用`runtime_Semrelease` 方法直接将当前锁交给下一个正在正在尝试获取锁的等待者,等待者会在被唤醒之后设置`mutexLocked`状态,由于此时仍然处于`mutexStarving`,所以新的Goroutine也无法获得锁
- 在正常模式下,如果当前互斥锁不存在等待者或者最低三位表示的状态都是0, 那么当前方法就不需要唤醒其他Goroutine可以直接返回,当有 Goroutine 正在处于等待状态时,还是会通过`runtime_Semrelease` 唤醒对应的 Goroutine并移交锁的所有权

<a name="25f9c7fa-1"></a>
##### 总结

- 如果互斥锁处于初始化状态,就会直接置位`mutexLocked`加锁
- 如果互斥锁处于`mutexLocked` 并且在普通模式下工作,就会进入自旋,执行30次`PAUSE`指令消耗CPU时间等待锁的释放
- 如果当前Goroutine等待锁的时间超过了1ms,互斥锁就会被切换到饥饿模式
- 互斥锁在正常情况下会通过`runtime_SemacquireMutex`方法将调用`Lock`的 Goroutine 切换至休眠状态,等待持有信号量的Goroutine唤醒当前协程
- 如果当前Goroutine 是互斥锁上的最后一个等待的协程或者等待的时间小于1ms,当前的Goroutine会将互斥锁切回正常模式
- 如果互斥锁已经被解锁,那么调用`Unlock`会直接抛出异常
- 如果互斥锁处于饥饿模式,会直接将锁的所有权交给队列中的下一个等待者,等待者会负责设置`mutexLocked`标志位
- 如果互斥锁处于普通模式,并没有Goroutine等待锁的释放或者已经有被唤醒的Goroutine获得了锁就会直接返回,在其他情况下会通过`runtime_Semrelease`唤醒对应的Goroutine

<a name="RWMutex"></a>
#### RWMutex

-  读写互斥锁在Go语音中的实现是 `RWMutex`,其中不仅包含一个互斥锁,还持有两个信号量,分别用于写等待读和读等待写 
```go
type RWMutex struct {
  w           Mutex
  writerSem   uint32
  readerSem   uint32
  readerCount int32
  readerWait  int32
}
Copy
```
 

-  `readerCount` 存储了当前正在执行的读操作的数量,最后的`readerWait`表示当写操作被阻塞时等待的读操作个数 

<a name="829abe8e"></a>
##### 读锁

- 读锁的加锁非常简单,通过`atomic.AddInt32`方法为`readerCount`加一,如果该方法返回了负数说明当前有Goroutine获得了写锁,当前Goroutine就会调用`runtime_SemacquireMutex`陷入休眠等待唤醒
- 如果没有写操作获取当前互斥锁,当前方法就会在`readerCount`加一后返回,当Goroutine想要释放读锁时会调用`RUnlock`方法,该方法会减少正在读资源的`readerCount`,当前方法如果遇到了返回值小于零的情况,说明有一个正在进行的写操作,在这时就应该通过`rUnlockSlow`方法减少当前写操作等待的读操作数`readerWait`并在所有都被释放之后出发写操作的信号量`writerSem`,`writerSem`被触发之后,尝试获取读写锁的进程就会被唤醒并获得锁

<a name="90109d41"></a>
##### 读写锁

- 当资源的使用者想要获取读写锁时,就需要通过`Lock` 方法了,在`Lock` 方法中首先调用了读写互斥锁持有的`Mutex`的`Lock`方法保证其他获取读写锁的Goroutine 进入等待状态,随后的`atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders)` 其实是为了堵塞后续的读操作
- 如果当时仍然有其他Goroutine持有互斥锁的读锁,该Goroutine就会调用`runtime_SemacquireMutex`进入休眠状态,等待读锁释放时触发`writerSem`信号量将当前协程唤醒
- 对资源的读写操作完成之后就会将通过`atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)` 变回正数并通过for循环出发所有由于获取读锁而陷入等待的`Goroutine`
- 最后,`RWMutex` 会释放持有的互斥锁让其他的协程能够重新获取读写锁

<a name="25f9c7fa-2"></a>
##### 总结

- `readerSem` - 读写锁释放时通知由于获取读锁等待的Goroutine
- `writerSem` - 读锁释放时通知由于获取读写锁等待的Goroutine
- `w`互斥锁 - 保证写操作之间的互斥
- `readerCount` - 统计当前进行读操作的协程数,触发写锁时会将其减少`rwmutexMaxReaders`阻塞后续的读操作
- `readerWait` - 当前读写锁等待的进行读操作的协程数,在出发`Lock`之后的每次`RUnlock`都会将其减一,当它归零时该Goroutine就会获得读写锁
- 当读写锁被释放`Unlock`时首先通知所有的读操作,然后才会释放持有的互斥锁,这样能够保证读操作不会被连续的写操作`饿死`

<a name="WaitGroup"></a>
#### WaitGroup

-  `WaitGroup` 是Go语言`sync` 包中比较常见的同步机制,它可以用于等待一系列的Goroutine的返回,一个比较常见的使用场景是批量执行RPC或者调用外部服务 
-  通过`WaitGroup` 我们可以在多个Goroutine之间非常轻松的同步信息,原本顺序执行的代码也可以在多个Goroutine中并发执行,加快了程序处理的速度 
```go
type WaitGroup struct {
  noCopy noCopy

  state1 [3]uint32
}
Copy
```
 

-  `noCopy` 的主要作用就是保证`WaitGroup` 不会被开发者通过再赋值的方式进行拷贝,进而导致一些诡异的行为 
-  `copyLock` 包就是一个用于检测类似错误的分析器,它的原理就是在 编译期间 检查被拷贝的变量中是否包含的`noCopy` 或者`sync` 关键字 
-  除了`noCopy`之外,`WaitGroup` 结构体中还包含一个总共占用12字节大小的数组,这个数组中会存储当前结构体持有的状态和信号量,在64位与32位机器上表现也非常不同 
-  `WaitGroup`提供了私有方法 `state`能够帮助我们从`state1`字段中取出它的状态和信号量 
-  `WaitGroup` 对外暴露的接口只有三个`Add`,`Wait`和`Done`,其中`Done` 只是调用了`wg.Add(-1)` 
-  `Add`方法的主要作用就是更新`WaitGroup`中持有的计数器`counter`,64位状态的高32位,虽然`Add`方法传入的参数可以为负数,但是一个`WaitGroup`的计数器只能是非负数,当调用`Add`方法导致计数器归零并且还有等待的Goroutine时,就会通过`runtime_Semrelease` 唤醒处于等待状态的所有Goroutine 
-  另一个`WaitGroup`的方法`Wait`就会在当前计数器中保存的数据大于0 时修改等待Goroutine 的个数`waiter`并调用 `runtime_Semacquire`陷入睡眠状态 
-  陷入睡眠的`Goroutine` 就会等待`Add`方法在计数器为 0 时唤醒 

<a name="25f9c7fa-3"></a>
###### 总结

- `Add`不能在和`Wait` 方法在 Goroutine 中并发调用,一旦出现就会造成程序崩溃
- `WaitGroup` 必须在 `Wait` 方法返回之后才能被重新使用
- `Done` 只是对`Add` 方法的简单封装,我们可以向`Add`方法传入任意负数(需要保持计数器非负)快速将计数器归零以唤醒其他等待的Goroutine
- 可以同时有多个Goroutine等待当前`WaitGroup` 计数器的归零,这些Goroutine也会被同时唤醒

<a name="Once"></a>
#### Once

-  
```go
Once
```
<br />保证在Go程序运行期间 <br />对应的某段代码只会执行一次  

-  作为`sync`包中的结构体,`Once`有着非常简单的数据结构,每一个`Once`结构体都只包含一个用于标识代码块是否被执行过的`done`以及一个互斥锁`Mutex` 
-  `Once` 结构体对外唯一暴露的方法就是`Do`,该方法会接受一个入参为空的函数,如果使用`atomic.LoadUint32`检查到已经执行过函数了,就会直接返回,否则会进入`doSlow`运行传入的函数 
-  `doSlow`的实现也非常简单,我们先为当前的Goroutine获取互斥锁,然后通过`defer`关键字将`done` 成员变量设置成1 并运行传入的函数,无论当前函数是正常运行还是抛出`panic`,当前方法都会将`done`设置成1 保证函数不会执行第二次 

<a name="25f9c7fa-4"></a>
##### 总结

- `Do`方法中传入的函数只会被执行一次,哪怕函数中发生了`panic`
- 两次调用`Do`方法传入不同的函数时只会执行第一次调用的函数

<a name="Cond"></a>
#### Cond

-  Go语言在标准库中提供的`Cond` 其实是一个条件变量,通过`Cond`我们可以让一系列的 Goroutine 都在触发某个事件或者条件时才被唤醒,每一个`Cond`结构体都包含一个互斥锁`L` 
```go
type Cond struct {
  noCopy noCopy

  L Locker

  notify  notifyList
  checker copyChecker
}
Copy
```
 

-  `Cond` 结构体中包含`noCopy` 和 `copyChecker` 两个字段,前者用于保证`Cond` 不会再编译期间拷贝,后者保证在运行期间发生拷贝会直接`panic`,持有的另一个锁`L`其实是一个接口`Locker`,任意实现`Lock`和`Unlock`方法的结构体都可以作为`NewCond`方法的参数 
-  结构体中的最后的变量`notifyList` 其实也就是为了实现`Cond`同步机制,该结构体其实就是一个Goroutine的链表 
-  `Cond` 对外暴露的`Wait` 方法会将当前Goroutine陷入休眠状态,它会先调用`runtime_notifyListAdd`将等待计数器 +1,然后解锁并调用 `runtime_notifyListWait` 等待其他Goroutine的唤醒 
-  `notifyListWait` 方法的主要作用就是获取当前的Goroutine并将它追加到 `notifyList`链表的最末端 
-  除了将当前Goroutine追加到链表的最末端之外,我们还会调用`goparkunlock`陷入睡眠状态,该函数也是在Go语音切换Goroutine 时经常会使用的方法,它会直接让当前处理器的使用权并等待调度器的唤醒 
-  `Cond`对外提供的`Signal` 和 `Broadcast` 方法就是用来唤醒调用`Wait`陷入休眠的Goroutine,前者会唤醒队列最前面的Goroutine,后者会唤醒队列中全部的Goroutine 
-  `notifyListNotifyAll`方法会从链表中取出全部的Goroutine并为他们依次调用 `readyWithTime`, 该方法会通过`goready` 将目标的Goroutine 唤醒 
-  虽然它会唤醒全部的Goroutine,但是这里唤醒的顺序其实还是按照加入队列的先后顺序,先加入的会先被`goready`唤醒,后加入的Goroutine可能就需要等待调度器的调度 
-  `notifyListNotifyOne` 函数就只会从 `sudog` 构成的链表中满足 `sudog.ticket == l.notify`的Goroutine并通过`readyWithTime`唤醒 
-  在一般情况下我们会选择在不满足特定条件时调用`Wait`陷入休眠,当某些Goroutine检测到当前满足了唤醒的条件,就可以选择使用`Signal`通过一个或者 `Broadcast`通知全部的Goroutine当前条件已经满足,可以继续完成工作了 

<a name="25f9c7fa-5"></a>
##### 总结

- 与`Mutex` 相比,`Cond`还是一个不被所有人都清楚和理解的同步机制,它提供了类似队列的FIFO的等待机制,同时也提供了`Signal` 和 `Broadcast` 两种不同的唤醒方法,相比于使用`for{}` 忙碌等待,使用`Cond`能够在遇到长时间条件无法满足时将当前处理器让出的功能,如果我们合理使用还是能够在一些情况下提升性能
- `Wait`方法在调用之前一定要使用`L.Lock` 持有该资源,否则会发生`panic`导致程序崩溃
- `Signal`方法唤醒Goroutine都是队列最前面 等待最久的Goroutine
- `Broadcast`虽是广播通知等待的Goroutine,但是真正被唤醒时也是按照一定顺序的
