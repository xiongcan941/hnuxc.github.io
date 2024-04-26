# Golang数据结构

https://bytedance.larkoffice.com/wiki/wikcn0t9vCu8TqhVCTdAHBkUFqc

## Map

特性：
    健值对存储

    独一的键

    通过键取回值

    键应该具有可比性（键不能为maps,slice,func)

golang的桶

    每个桶有八个槽位用于存储健值对数据

    如果我们需要多于8个槽位，overflow指针灰指向另一个桶（已弃用）

    top hash(e0-e7)，比如e0就是key0的哈希值取高8位，主要的目的是加速比较，用于查看要查找的key是否在桶中，加速比较

    超过128字节的键值对将被转化为指针

如果key和value都不包含指针并且可以内联（<=128字节），则hmap的额外字段extra用于存储溢出桶，可以避免GC（垃圾回收）扫描整个map（因为要扫描保证对应的指针是活着的）

```
type hmap struct {
    // Note: the format of the hmap is also encoded in cmd/compile/internal/reflectdata/reflect.go.
    // Make sure this stays in sync with the compiler's definition.
    flags     uint8
    noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
    hash0     uint32 // hash seed

    count     int // # live cells == size of map.  Must be first (used by len() builtin)
    B          uint8  // 决定桶的数量log_2 of # of buckets (can hold up to loadFactor * 2^B items)
    buckets    unsafe.Pointer // 指向桶数组的指针array of 2^B Buckets. may be nil if count==0.
    oldbuckets unsafe.Pointer // 如果触发扩容会用到这个字段previous bucket array of half the size, non-nil only when growing
    nevacuate  uintptr        // 扩容时使用，表示已经有多少个桶迁移到新的桶去了progress counter for evacuation (buckets less than this have been evacuated)

    extra *mapextra // optional fields
}
```

### 查询操作lookup

    v=m[key] 会被编译成v=runtime.lookup(m,k)

golang的map不支持泛型，而是利用unsafe pointer，可以指向任意的类型，跟c中（void *)差不多。但是会有辅助的类型说明key,value是什么类型

```
type mapType struct{
    key       *_type
    value     *_type
每种类型都有对应的equal方法和hash方法，equal方法用于判断是否有两个key相等，hash方法用于
对key进行哈希
type _type struct{
    size      uintptr
    equal     func(unsafe.Pointer,unsafe.Pointer) bool
    hash      func(unsafe.Pointer,uintptr) uintptr
}
```

底层逻辑

```
v=m[k]
被编译成
pk:=unsafe.Pointer(&k)
pv:=runtime.lookup(typeOf(m),m,pk) // pv:unsafe.Pointer
v=*(*V)pv//先将pv转换成对应的指针类型，再解引用
```

```
func lookup(t *mapType,m *hmap,k unsafe.Pointer) unsafe.Pointer{
    if m==nil || m.count==0{
        return zero
    }
    // get hash of key
    hash:=t.key.hash(k,m.seed)
    bucket:=hash & (1<<m.B-1) // e.g. 8 buckets,B为3，(1<<m.B-1)为111, the last 3-bits of hash is the bucket
    extra:=byte(hash>>56) // extra:the top 8 bits of hash，哈希值共64位
找到对应桶位置,m.buckets表示桶数组基地址，t.bucketssize表示该类型对应的桶大小
，再乘上bucket就是对应桶地址    b:=
(*bucket)add(m.buckets,bucket*t.bucketssize) // address of the buckets

//进入到桶中
    for{
        for i:=0;i<8;i++{
            if b.extra[i]!=extra{
                continue
            }
            //高8位相等后进一步比较key是否相等
            key:=add(b,dataOffset+i*t.key.size) // point to keyi in bucket
            if t.key.equal(key,k){//key存的原值，哈希值只用前8位匹配一下
                return add(b,dataOffset+8*t.key.size+i*t.value.size)//定位到对应
值的地址，即桶的起始地址b加上dataOffset加上八个键的偏移8*t.key.size加上i个值的偏移i*t.value.size      
            }
        }
        b=b.overflow //在第一个桶里没有找到，则到overflow对应的桶继续找
        if b==nil{
            return zero//直到找完，没有找到
        }
    }
}
```

### 插入与删除

基本与lookup一致，但是可能触发扩容，在插入修改与删除之前判断map是否太满，如果太满，我们需要扩容map

(1)太满的条件：

            平均每个桶里都含有超过6.5个健值对

            overflow的桶太多了，如果这个太多了的话，搜索的时候将会退化成链表：当桶总数<2^15时，如果溢出桶总数>=桶总数，则认为溢出桶过多。 当桶总数>=2^15时，直接与2^15比较。桶总数是那个桶数组的数量，即不包含溢出桶的桶数量

- Growing steps:
1. Allocate a new array of buckets of twice the size在内存中分配两倍数量的桶空间
2. Copy entries over from the old buckets to the new buckets将全部旧桶复制到新桶中，释放掉旧桶，拷贝操作比较耗时，采用逐步扩容的方式，即如果写入0号桶，但是正在扩容，则将0号桶迁移过来（写时复制？？？）
3. Use the new buckets       
   - 扩展后某些key可能在另一个bucket中
     - 只有2种可能：
       - 扩展前：4个桶，桶的索引由哈希的最后2位决定：
         - key0 哈希=0010 个桶=2，key1 哈希=0100 个桶=0
       - 扩展后：8个桶，桶的索引由哈希的最后3位确定：
         - key0 哈希=0010 桶=2（保持不变），key1 哈希=0100 桶=4

### 遍历

遍历map时，并不是从固定的bucket 0开始，每次遍历都是从一个有随机个值的bucket开始，然后从随机的cell开始遍历。 然后它按桶顺序遍历它，直到返回到起始桶的末尾。

为了保证遍历map的结果是乱序的？？？

### 比较

c++允许&m[k]，取值的地址，但是golang不允许，因为扩容后会导致地址失效，则编译的时候就会报错

# 
