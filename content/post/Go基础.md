---
title: "Go基础"
date: 2020-03-20T17:41:16+08:00
draft: false
---
# Go基础

- := 自动推导类型, 只能在第一次 生成时候使用

- Println 自动加换行符
  Printf 格式化输出 

  - ![1552268553178](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1552268553178.png)
  - ![1552268572930](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1552268572930.png)

- 支持多重赋值写法
   i, j := 20, 30
     	i, j = j, i

- _ 匿名变量, 丢弃数据不处理

- 变量:
  程序运行期间, 可以改变的量, 变量声明需要var
    	var	(多个变量的声明)
  常量:
  	程序运行期间, 不可以改变的量, 常量声明需要const
  	const (多个常量的声明)

- iota:
  1. iota 常量自动生成器, 每个一行, 自动累加1
    2. iota 给常量赋值使用
    3. iota遇到const, 重置为0
    4. 一个const里面,可以只写一个iota, 下面省略
    5. 同一行的iota, 值都一样

- 数据类型

  - 布尔类型 默认值false

  - 浮点型

    - float64比float32更准确

  - 字符

    - byte
      - 单引号
      - 大小写相差32, 小写大
    - string
      - 双引号
      - len()
      - 有个隐藏结束符 '\0'

  - 类型转换

    - bool类型不能转换为int, 整型不能转换为bool
    - 字符型本质上就是整型 int('a')

  - 类型别名

    - type bigint int64

  - 运算符

    - % 取余 
    - 优先级
      - ![1552269800808](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1552269800808.png)

  - 流程控制

    - if 条件 {}

      - if a := 10; a == 10 {

        }else {

        }

        // **注意**这里a的作用域 只有这个if 语句

    - switch..{case 条件: }

      - 默认包含break, 可以不写
      - fallthrough  不跳出switch, 后面无条件执行
      - 可以没有条件, 可以支持一个声明语句

    - 循环只有for

      ```go
      for i := 1; i <= 100; i++ {
          sum = sum + i
      }
      for {}
      // for 不加条件就是死循环
      ```

    - range

      ```
      for i , data := range str{}
      // i 接受下标, data接收值
      for i := range str{}
      // 丢弃值, 只接收下标
      ```

    -  goto  无条件跳转

  - 函数

    - go推荐写法给返回值一个变量名

      ```go
      func myfunc() (result int){
          result = 666
          return
      }
      ```

    - 函数名称首字母小写即为private, 大写就是公有函数public

    - 有返回值的函数必须通过return返回

    - 函数类型-->多态

    - 闭包

      - 一个函数捕获了和他在同一作用域的其他常量和变量

      - ```go
        func test() func() int {
            var x int
            return func() int{
                x++
                return x * x
            }
        }
        
        func main(){
            // 返回值为一个匿名函数, 返回一个函数类型, 通过f 来调用返回的匿名函数, f来调用闭包函数
            // 他不关心这些捕获了的变量和常量是否已经超出了作用域
            // 所以只要闭包还在使用它, 这些变量就还会存在
            f := test()
            fmt.Println(f())	//1
            fmt.Println(f())	//4
            fmt.Println(f())	//9
            fmt.Println(f())	//25
        }
        ```

    - defer

      - 延迟调用, main 函数结束之前调用
      - 多个defer语句, 会以LIFO(后进先出)的顺序执行, 哪怕函数或某个延迟调用发生错误, 这些调用依旧会被执行
        - 报错在最后 !
      -  参数会保留经过defer 时候的值, 不受后面修改影响

    - 获取命令行参数

      ```go
      import {
          "os"
      }
      
      func main(){
          args := os.Args	//获取用户输入的所有参数
      }
      ```

    - 局部变量, 只在其定义的{}内有效. 执行到定义变量那句话, 才开始分配空间, 离开作用域自动释放

    - 全局变量, 定义在函数外部的变量是全局变量 

    - 不同作用域, 允许定义同名变量

      - 函数内部有, 优先使用函数内部的, 函数内部没有, 使用全局变量

  - 工程管理

    - Go代码必须放在工作区中

      - src目录: 用于以代码包的形式组织并保存Go源码文件(比如: .go .c .h .s等)
      - pkg目录: 用于存放经由go install 命令构建安装后的代码包(包含Go库源码文件)的".a"归档文件
      - bin目录: 与pkg目录类似, 在通过go install 命令完成安装后, 保存由Go命令源码文件生成的可执行文件.

    - 导包:

      ```go
      //写法1
      import "fmt"
      import "os"
      // 写法2
      import(
      	"fmt"
          "os"
      )
      // 导入所有
      import . "fmt"
      // 给包起别名
      import io "fmt"
      // 忽略导入的包, 只使用其中的init函数
      import _ "fmt"
      
      ```

    - go env 查看环境

    - 同一个目录, 调用别的文件的函数, 直接调用即可, 无需包名引用

      不同目录, 调用不同包里面的函数, 格式: 包名.函数名()

       - 注意函数名小写是不能被调用到的
       - 必须首字母大写

    - ![1552297824113](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1552297824113.png)

    - go install 自动生成bin 或 pkg 目录

      - 除了要配置GOPATH, 还要配置GOBIN

  - 复合类型

    - 指针

      - ```go
        package main
        
        func main(){
            var a int
            //每个变量有两层含义: 内存, 地址
            fmt.Printf("a=%d\n", a)
            fmt.Printf("&a = %v\n", &a)
        }
        ```

      - 定义一个变量 类型为 *int

        ```go
        var p * int
        p = &a	//指针变量指向谁, 就把谁的地址赋值给指针变量
        *p = 666 //*p 操作的不是p的内存, 是p所指向的内存(a)
        p = new(int)  // p是*int, 指向int类型, 创建一片合法内存
        ```

      - 默认值 ``nil``, 没有NULL常量

      - 操作符 & 取地址,  * 通过指针访问目标

      - 不支持 ->

    - 数组

      - var id [40]int

      - 声明变量的同时赋值, 叫初始化

        - 1. 全部初始化

             ```go
             var a [5]int = [5]int{1,2,3,4,5}
             b := [5]int{1,2,3,4,5}
             ```

          2. 部分初始化, 没有初始化的元素, 自动赋值为0

      - 二维数组

        - 有多少个[]就是多少维
        - 有多少个[]就用多少个循环

      - 两个数组比较, 只支持 == 或 !=.   每个元素都相同才相等

    - 随机数

      ```go
      import "math/rand"
      import "time"
      func main(){
          
          rand.Seed(666)
          rand.Seed(time.Now().UnixNano()) //以系统时间作为种子
          for i := 0; i < 5; i++{
              rand.Int()  //随机很大的数
              rand.Intn()		//随机100以内的数
          }
      }
      ```

    - 切片的长度和容量

      - [low: high : max]

        左闭右开

        len = high - low

        cap = max - low

      - 切片和数组的区别

        - 数组[]里面的长度是固定的一个常量, 数组不能修改长度, len 和 cap 固定
        - 切片, [ ] 里面为空或者为...    ,   切片的长度或容量可以不固定

      - 切片的创建

        ```go
        //自动推导类型
        s1 := []int{1,2,3,4}
        //借助make函数, 格式  make(切片类型, 长度, 容量)
        s2 := make([]int, 5, 10)
        //容量可以不写. 没有指定容量, 那么容量和长度一样
        ```

      - 切片的截取

      - 切片的内建函数

        - append
          -  在末尾添加元素
          - 如果超过原来的容量, 通常以两倍容量扩容
        - copy
          - copy(dstSlice, srcSlice)
          - 把srcSlice 覆盖到dst的相同位置

    - map

      - map 只有len ,没有cap

      - 删除key为1的内容

        ```go
        delete(m, 1)
        ```

      - m := make(map[int]string, cap)

    - 结构体

      ```go
      type Student struct{
          id 		int
          name 	string
          sex		byte
      }
      ```

  - 面向对象

    - 特性

      ​	封装: 通过方法实现

      ​	集成: 通过匿名字段实现

      ​	多态: 通过接口实现

    - 匿名字段

      ```go
      type Person struct {
          name string
          sex byte
          age int
      }
      type Student struct{
          Person	// 只有类型, 没有名字, 匿名字段, 继承Person的成员
          id int
          addr string
      }
      func main(){
          // 顺序初始化
          var s1 Student = Student{Person{"mike", "m", 18}, 1, "bj"}
          // %+v  输出键和值 
          fmt.Printf("s1 = %+v\n", s1)
      
      }
      ```

    - 就近原则: 

      - 当有同名字段时候, 优先操作本作用域的字段
      - 先找本作用域的方法, 找不到再用继承的方法

    - 方法:

      - 只要接收者类型不一样, 这个方法就算同名也是不同的方法, 不会出现重复定义函数的错误

      - 指针变量方法和普通变量方法可以通用, 叫做方法集

      - 方法可以继承

        - 先找本作用域的方法, 找不到再用继承的方法
        - 显式调用继承的方法``s.Person.PrintInfo()``

      - 方法值

        ```go
        p.SetInfoPointer()
        pFunc := p.SetInfoPointer	//这个就是方法值, 调用函数时, 无需再传递接收者, 隐藏了接收者
        pFunc()		//等价于 p.SetInfoPointer()
        ```

    - 接口

    - 类型断言

      ```go
      i := make([]interface{}, 3)
      i[0] = 1
      i[1] = "hello go"
      i[2] = Student{"mike", 666}
      //类型查询
      for index, data := range i{
          if value, ok := data.(int); ok == true {
              fmt.Printf("x[%d] 类型为int, 内容为 %d \n", index, value)
          }
      }
      
      //通过switch实现
      for index, data := range i{
          switch value := data.(type){
              case int:
              	fmt.Printf()
              case string:
              	fmt.Printf()
              case Student:
              	fmt.Printf()        	               	
          }
      }
      ```

  - 异常处理

    - package errors

    - ```go
      if err != nil{
          fmt.Println("err= ", err)
      }
      ```

    - panic  致命错误

    - ```go
      func testb(x int){
          defer func(){
              //revocer()	//可以打印panic的错误信息
              //fmt.Println(recover())
          }
      }
      ```

    - recover

      - recover 只有在defer 调用的函数中有效
      - recover可以使程序从panic中恢复, 并返回panic value.导致panic异常的函数是不会

  - 文本文件处理

    - 字符串处理

      ```go
      string.Contains(s, sub string)
      	字符串s中是否包含substr,返回布尔值
      string.Join(s, ",")
      	合并数组s := []string{"foo", "bar", "baz"}
      string.Index("chicken", "ken")
      	查找位置, 返回int 4, 没有返回 -1
      string.Repeat("na", 2)
      	重复2次, 返回生成的字符串
      string.Replace("oink oink oink", "k", "ky", 2)
      	string.Replace(s, old, new string, n)
      	在s字符串中, 把old字符串替换为new, n表示替换次数, 负数表示全部替换
      string.Split(s, sep)
      	把s 按照sep分割, 返回slice, 不包含sep
      string.Trim (s, cutset)
      	去除s里面左右两边的的cutset字符串, 返回去除之后的串
      string.Field(s)	
      	去除 s 字符串的空格符, 并且按照空格分割返回slice
      ```

    - 字符串转换

      ```go
      追加, 字节数组方法, slice := make([]byte, 0, 1024)
      	strconv.AppendInt(str, 4567, 10) //以10进制方式追加
      	strconv.AppendBool(str, false)
      	strconv.AppendQuote(str, "ablceffe")
      	strconv.AppendQuoteRune(str, '单')
      其他类型转换为字符串
      	strconv.Format(false)
      	strconv.FormatFloat(3.14, 'f', -1, 64)
      	strconv.Itoa(6666)
      字符串转其他类型
      	flag, error = strconv.ParseBool("false")
      ```

    - 正则表达式

      - import "regexp"

      - 使用

        ```go
        import "regexp"
        
        func main(){
            buf := "abc azc a7c 888"
            // 1. 先编译规则, 解析正则表达式, 编译成功返回一个解释器
            reg1 := regexp.MustCompile('a.c')
            if reg1 == nil {
                //解释失败, 返回nil
                fmt.Println("regexp err")
                return
            }
            //2. 根据规则提取信息
            result1 := reg1.FindAllStringSubmatch(buf, -1) //-1表示全部匹配
            fmt.Println("result = ", result1)	//[[abc] [azc] [a7c]]
        }
        ```

    - json

      - 生成

        1. 结构体生成

           ```go
           func Marshal(v interface{}) ([]byte, error)
           
           //成员变量名首字母必须大写
           type IT struct{
               Company string	'json: "company"'	//二次编码
               Subjects []string	'json:"-"'	//此字段不会输出到屏幕
               IsOk	bool	'json:",string"'	//编码成
           }
           func main(){
           	//定义一个结构体变量
               s := IT{"itcast", []string{"GO", "Python"}, true}
               //编码, 根据内容生成json文本
               buf, err := json.Marshal(s)
               //格式化编码
               // buf, err := json.MarshalIndent(s, "", " ")
               if err != nil {
                   fmt.Println("err = ", err)
                   return
               }
               fmt.Println("buf = ", string(buf))
           }
           
           ```

           

        2. map生成

           ```go
           func main(){
               //创建一个map
               m := make(map[string]interface{}, 4)
               m["company"] = "itcast"
               m["subjects"] = []string{"GO", "C++", "Python", "Test"}
               m["isok"] = true
               
               //编码成json
               result, err := json.Marshal(m)
               if err != nil{
                   fmt.Println("err = ", err)
                   return
               }
               fmt.Println("result = ", string(result))
           }
           ```

      - 解析json

        1. 解析到结构体

           ```go
           func main(){
               jsonBuf := `
           	{
           	"company": " itcast",
               "subjects": [
               	"GO",
               	"Python",
               	"C++"
               ],
               "isok": true
           	}
           	`
               var tmp IT	//定义一个结构体变量
               err := json.Unmarshal([]byte(jsonBuf), &tmp) 	//第二个参数要地址传递
               if err != nil{
                   return
               }
               fmt.Println("tmp = ", tmp)
           }
           ```

           

        2. 解析到map

           ```go
           func main(){
               m := make(map[string]intrerface{}, 4)
               //需要使用断言来反推类型, 不能直接转换类型
           }
           ```

    - 文件的读写

      ```go
      func WriteFile(path string){
          // 新建文件
          f, err := os.Create(path)
          if err != nil {
              fmt.Println("err = ", err)
              return
          }
          //使用完毕, 需要关闭文件
          defer f.close()
          
          for i := 0; i < 10; i++{
              buf = fmt.Sprintf("i = %d\n", i)
              n, err := f.WriteString(buf)
              if err != nil {
                  fmt.Println("err = ", err)
              }
          }
      }
      ```

      读文件

      ```go
      func ReadFile(path string){
          f, err = os.Create(path)
          if err != nil {
              fmt.Println("err = ", err)
              return
          }
          def f.close()
          buf := make([]byte, 1024.2) 	//2K大小
          n, err1 := f.Read(buf)
          if err1 != nil {
              fmt.Println("err1 = ", err1)
              return
          }
          fmt.Println("buf = ", string(buf[:n]))
      }
      ```

      按行读取

      ```go
      f, err := os.Open(path)
      defer f.close()
      r := bufio.NewReader(f)
      for {
          \\遇到换行停止读取, 每行包含\n
          buf, err := r.ReadBytes('\n')
          if err != nil{
              if err == io.EOF{
                  break
              }
              fmt.Println("err = ", err)       
          }
          fmt.Printf("buf = %s\n", string(buf))
      }
      
      ```

    - 拷贝文件

      ```go
      //只读方式打开源文件
      sF, err := os.Open(srcFileName)
      if err != nil {
          fmt.Println("err = ", err)
          return
      }
      dF, err2 := os.Create(srcFileName)
      if err2 != nil {
          fmt.Println("err2 = ", err2)
          return
      }
      //操作完毕, 需要关闭文件
      defer sF.Close()
      defer dF.Close()
      
      //拷贝文件
      buf  := make([]byte, 4*1024)
      for {
          n , err := sF.Read(buf) 
          if err != nil {
              if err == io.EOF{
                  break
              }
              fmt.Println("err= ", err)        
          }
          //往目的文件写, 读多少写多少
          df.Write(buf[:n])
      }
      ```

  - 并发编程

    - goroutine

      - 只需在函数调用语句前添加go关键字, 就可以创建并发执行单元.

      - Gosched()

        - 让出时间片, 先让别的协程执行

        - import  "runtime"

          runtime.Gosched()

      - Goexit()

        - 终止所在的协程

      - GOMAXPROCS()

        - 设置可以并行计算的CPU核数的最大值, 并返回之前的值

    - channel

      - var ch = make(chan int)

      - 无缓冲的channel, 在接收前没有能力保存任何值的通道

        ``ch := make(chan int 0)``

      - 有缓冲的channel

        - 只有在通道全满或全空时候阻塞

      - 关闭channel

        ```go
        ch := make(chan int)	//创建一个无缓存的channel
        //不需要写数据时, 关闭channel
        close(ch)
        ```

        关闭channel后, 可以继续读, 不能发, 会引发panic, 导致接收立即返回零值

      - 单向的channel

        ```go
        var ch1 chan int 	//双向channel
        var ch2 chan <- float64		//只用于写float64数据
        var ch3 <- chan int 	//只用于读取int数据
        ```

        生产者, 消费者

    - 定时器

      - Timer

        - 一个定时器, 代表未来一个单一事件, 可以告诉timer你要等待多长时间, 它提供一个channel, 在将来的那个时间那个channel提供了一个时间值

        ```go
        import "time"
        func main(){
            timer := time.NewTimer(2* time.Second)
            //2s后, 有数据可以读取
            t := <- time.C	//channel没有数据阻塞    
        }
        ```

      - Ticker

        计时器

        ```go
        ticker := time.NewTicker(2 * time.Second)
        ```

    - select

      - 监听channel上面的数据流动

      - 与switch语句类似, 但是**每个case语句里必须是一个IO操作**

        ```go
        select {
            case <- chan1:
            	//如果chan1成功读到数据
            case chan2 <- 1:
            	// 如果成功向chan2写入数据  
            default:
            	//如果case里面的情况都不满足, 就执行default
            	//如果没有default语句,select阻塞, 指导有一个case满足
        }
        ```

      - 利用select实现超时

        ```go
        func main(){
            ch := make(chan int)
            go func(){
                for {
                    select {
                        case num := <-ch:
                        	fmt.Println("num =", num)
                        case <- time.After(3 * time.Second):
                        	fmt.Println("超时")
                        	quit <- true
                        
                    }
                }
            }()		//别忘了()
            
            for i := 0; i < 5; i++ {
                ch <- i
                time.sw
            }
            <-quit
            fmt.Println("程序结束")
        }
        ```

  - 网络编程

    - 每层协议的功能

      - 链路层: 设备到设备(MAC)
      - 网络层: 主机到主机(IP)
      - 传输层:进程到进程(端口号)
      - 应用层: 应用程序(json)

      ![1552902107846](E:\文档\笔记\Go基础.assets\1552902107846.png)

    - socket 编程

      - TCP服务器

      ```go
      package main
      
      import (
      	"fmt"
          "net"
      )
      
      func main(){
          // 监听
          listener, err := net.Listen("tcp", "127.0.0.1:8080")
          if err != nil {
              fmt.Println("err = ", err)
              return
          }
          defer listener.close()
          //阻塞等待
          conn, err := listener.Accept()
          if err != nil {
              fmt.Println("err = ", err)
              return
          }
          //接收请求
          buf := make([]byte, 1024)	//1024大小的缓冲区
          n, err1 := conn.Read(buf)
          if err1 != nil{
              fmt.Println("err1 = ", err1)
              return
          }
          fmt.Println("buf = ", string(buf[:n]))
          defer conn.Close()
          
      }
      ```

      - TCP客户端

      ```go
      package main
      
      import (
      	"fmt"
          "net"
      )
      
      func main(){
          //主动连接服务器
          conn, err := net.Dial("tcp", "127.0.0.1:8000")
          if err != nil {
              fmt.Println("err=", err)
              return
          }
          
          defer conn.Close()
          //发送数据
          conn.Write([]byte("are u ok?"))
      }
      ```

      - os.Stat(fliename)

        返回文件信息, 包括Name Size等

        

