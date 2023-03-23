
## 逻辑题
1.  A
2.  B
3.  C
4.  B
5.  A

## 语言特性
1.  [0 0 1 2]
2.  A B
3.  C
4.  C
5.  B

## 计算机
1.  3  4
2.  
    Cookie是存储在客户端浏览器中的文本文件，通过在客户端与服务端之间传递标识符实现用户身份的识别和验证；Session是存储在服务端的用户数据的一种机制，用于记录用户的会话信息。
3.  
    301：永久重定向
    302：临时重定向
    400：请求无效
    500：服务器内部错误
4. 
    grep: 用于在文件或标准输入中查找匹配指定模式的行。
top: 用于动态实时显示进程信息的实用程序。
find: 用于在指定目录下查找文件并执行指定操作。
mv: 用于移动或重命名文件和目录。
cat: 用于将一个或多个文件内容输出到标准输出。
du: 用于估算文件系统中文件的磁盘空间占用量。
wc: 用于统计文件中的字节数、字数、行数等信息。
chmod: 用于更改文件或目录的权限。
crontab: 用于管理系统中的定时任务。
awk: 用于在文件或标准输入中查找和处理文本。
uniq: 用于过滤或统计重复的文本行。
ps: 用于列出当前进程的快照。
5.  读写执行
6.  
    Explain: 用于分析MySQL查询语句的执行计划，包括使用了哪些索引、表的连接顺序等信息。
   Show processlist: 显示当前MySQL服务器的所有活动连接及其执行的SQL语句、执行状态、执行时间等信息。

7. 
   工厂模式
单例模式
适配器模式
装饰器模式
迭代器模式

## 算法与编程能⼒
1. 
```go
package main

import (
    "fmt"
)

func abs(x int) int {
    if x < 0 {
        return -x
    }
    return x
}

func minAbs(nums []int) int {
    n := len(nums)
    left, right := 0, n-1
    for left < right {
        mid := left + (right-left)/2
        if nums[mid] < 0 {
            left = mid + 1
        } else {
            right = mid
        }
    }
    if left == 0 {
        return nums[0]
    }
    if left == n {
        return nums[n-1]
    }
    if abs(nums[left-1]) < abs(nums[left]) {
        return nums[left-1]
    }
    return nums[left]
}

func main() {
    nums := []int{-10, -5, -3, 0, 3, 5, 7}
    fmt.Println(minAbs(nums))
}
```

2. 
```go
package main

import (
    "fmt"
)

func main() {
    fmt.Println(climbStairs(5))
}

func climbStairs(n int) int {
    if n == 1 || n == 2 {
        return 1
    }
    if n == 3 {
        return 2
    }
    f := make([]int, n+1)
    f[1], f[2], f[3] = 1, 1, 2
    for i := 4; i <= n; i++ {
        f[i] = f[i-1] + f[i-3]
    }
    return f[n]
}
```

