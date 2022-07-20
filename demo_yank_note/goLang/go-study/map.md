# map



#### map 三种声明方式

```go
// 第一种声明方式 var
// 使用map前，需要先使用make给map对象分配数据空间
// 空map赋值会 panic: assignment to entry in nil map
var myMap1 map[string]string
myMap1 == nil
myMap1 = make(map[string]string, 10)
```
```go
// 第二种声明方式 := make(map[string], int) 声明不初始化，常用
// make出来的map没有数据但不是nil
myMap2 := make(map[int]int)
myMap2 != nil
// map添加数据
myMap2[1] = 111
```
```go
// 第三种声明方式，声明并初始化， ：=推断，不需要make
myMap3 := map[string]string{
		"one":   "php",
		"two":   "java",
		"three": "python",
		"four": "go",
	}

```

#### map使用

```go
// map遍历使用
for key, value := range myMap3 {
		fmt.Println("key is ", key)
		fmt.Println("value is ", value)
	}
// 删除
delete(myMap3, "two")
//修改
myMap3["three"] = "go"
// 传参， 引用传递（指针传递）
func PrintMap(myMap map[string]string) {
	myMap["one"] = "go"
	for key, value := range myMap {
		fmt.Println("key is ", key)
		fmt.Println("value is ", value)
	}

}
```






