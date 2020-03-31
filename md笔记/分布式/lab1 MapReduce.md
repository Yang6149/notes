# lab1 MapReduce

听完 MIT6.824 第一节课后看到课后有实验，于是就赶忙过去写实验。看了一下实验要求，又滚回去看 paper。

## 总结

这次实验要求是使用 go 语言，然后尴尬的是我没有一点 go 语言的基础。但是这难不倒我，直接官方 quic start 走起。大概断断续续的花了两天的时间走完了一遍。（期间忙着复习以及准备面试笔试）

然后根据 paper 上的内容用 go 实现一遍，go 的语法一旦写起来感觉比 Java 爽多了。

期间从最开始上课，到最后写完大概用了十天左右吧，毕竟中间很多时间都在忙其他事情。隔两天来写一写这个，我觉得 15% 时间在学习 10% 时间在编码，剩下的时间全在 debug。

## lab 要求

lab 中提供了要使用的数据以及测试脚本。并且提供了一个串行实现方式。大概看看串行的方式就了解怎么回事了。自己只需要写`rpc`、`master`、`worker`这三个，然后通过 `mrmaster` 和`mrworker`调用自己写的函数。lab 中提供了一个建议是使用 json 来作为中间数据存储格式，并且也贴出了简易 demo 。最后的测试用例要求完成

* 和标准答案相同
* 为每个单词的所在文件通过单词归类（倒排索引）
* 测试 map 并发
* 测试 reduce 并发
* 测试容错

最后贴一下测试的结果吧，看着这些 pass 乐半天

![1585136791397](https://raw.githubusercontent.com/Yang6149/typora-image/master/demo/202003/25/194632-900736.png)

## 遇到的一些坑

### 语法

首先一点就是对语言的坑了，想要让别人也能访问的到参数，一定要大写首字母。这些还算好，因为在编译的时候会又提醒。最大的坑就是通过 rpc 调用的时候，传入的参数也要对这个大小写有效而且写为小写的化也不会报错更不会传入。

就因为这个我忙活了至少 6 个小时。

### master 和 worker 处理的逻辑

开始我想当然的在脑中构建信息传递逻辑，写也写出来了，运行 master 和单 worker 没毛病。问题就是在并发的状态下，老是处理错误，后来拿纸币重新画图，代码删完重写一遍才跑通。纸上画的更清楚，就不贴出来了。

![1585137003388](C:\Users\hasaki\AppData\Roaming\Typora\typora-user-images\1585137003388.png)

### 锁以及超时

由于开始我使用的是 master 单线程处理所有数据没有出现问题，重构之后使用了协程为每一次请求做出回应并设立超时，这样可以有效的保证容错，这里也需要加上锁保证安全。 

## 代码

master

```go
package mr

import "log"
import "net"
import "os"
import "net/rpc"
import "net/http"
import "strconv"
import "time"
import "sync"
import "context"
import "fmt"


const(
	isworking = 1
	idle = 2
	commited = 3


)
type Master struct {
	timeout time.Duration
	reduceN int
	mapN int
	mapWork []WorkStatus//
	reduceTable []WorkStatus
	mu sync.RWMutex
	allFinished bool



}
type WorkStatus struct{
	filename string
	status int// 1为完成 0 为未完成  2 等待
	start int64
}

// Your code here -- RPC handlers for the worker to call.
func (m *Master) Handler(args *MyRPCArgs, reply *MyRPCReplay) error {
	m.mu.Lock()
	defer m.mu.Unlock()

	if m.allFinished{
		reply.AllFinished=true;
		return nil
	}


	if args.Status=="apply"{
		reply.MapN=m.mapN
		reply.ReduceN=m.reduceN
		reply.AllFinished=m.allFinished
		//当分配完任务后直接返回，让worker退出
		if m.allFinished{
			return nil
		}
		//开始分配map任务
		for i :=range m.mapWork{
			if m.mapWork[i].status==commited||m.mapWork[i].status==isworking{
				continue
			}
			//当任务还没开始或任务超时时，分配任务
			if m.mapWork[i].status==idle||m.mapWork[i].status==0{
				m.mapWork[i].status=isworking
				reply.Type=1
				reply.Id=i
				reply.Filename=m.mapWork[i].filename
				//设立超时，当前任务超过十秒没有完成任务
				ctx, _ := context.WithTimeout(context.Background(), m.timeout)
				go func() {
					select {
					case <-ctx.Done():
						{
							m.mu.Lock()
							//再次判断如果任务没有完成则状态改为 dile
							if m.mapWork[i].status!=commited{
								fmt.Printf("超时了%v",i)
								m.mapWork[i].status=idle
							}
							m.mu.Unlock()
						}
					}
				}()
			}
			return nil
		}
		for i :=range m.mapWork {
			if m.mapWork[i].status == commited {
				continue
			}
			return nil
		}
		//接下来是reduce部分
		for i :=range m.reduceTable{
			if m.reduceTable[i].status==commited||m.reduceTable[i].status==isworking{
				continue
			}
			//当任务还没开始或任务超时时，分配任务
			if m.reduceTable[i].status==idle||m.reduceTable[i].status==0{
				m.reduceTable[i].status=isworking
				reply.Type=2
				reply.Id=i
				reply.Filename="mr-out-"+strconv.Itoa(i)
				//设立超时，当前任务超过十秒没有完成任务
				ctx, _ := context.WithTimeout(context.Background(), m.timeout)
				go func() {
					select {
					case <-ctx.Done():
						{
							m.mu.Lock()
							//再次判断如果任务没有完成则状态改为 dile
							if m.reduceTable[i].status!=commited{
								m.reduceTable[i].status=idle
							}
							m.mu.Unlock()
						}
					}
				}()
			}
			return nil
		}

	}
	if args.Status=="commit"{
		if args.Type==1{
			m.mapWork[args.Id].status=commited
		}else{
			m.reduceTable[args.Id].status=commited
		}

		bo := true
		for i:=range m.mapWork{
			if m.mapWork[i].status!=commited {
				bo=false
				break
			}
		}
		for i:=range m.reduceTable{
			if m.reduceTable[i].status!=commited {
				bo=false
				break
			}
		}
		m.allFinished=bo
		reply.AllFinished=bo
		return nil
	}
	return nil
}

//
// start a thread that listens for RPCs from worker.go
//
func (m *Master) server() {
	rpc.Register(m)
	rpc.HandleHTTP()
	//l, e := net.Listen("tcp", ":1234")
	sockname := masterSock()
	os.Remove(sockname)
	l, e := net.Listen("unix", sockname)
	if e != nil {
		log.Fatal("listen error:", e)
	}
	go http.Serve(l, nil)
}

//
// main/mrmaster.go calls Done() periodically to find out
// if the entire job has finished.
//
func (m *Master) Done() bool {


	// Your code here.

	return m.allFinished
}

//
// create a Master.
// main/mrmaster.go calls this function.
// nReduce is the number of reduce tasks to use.
//
func MakeMaster(files []string, nReduce int) *Master {

	mapWork:=make([]WorkStatus,len(files))
	for i:=range files{
		mapWork[i]=WorkStatus{files[i],0,0}
	}
	reduceTable:=make([]WorkStatus,nReduce)
	for i:=0;i<nReduce;i++{
		reduceTable[i]=WorkStatus{"mr-out-"+strconv.Itoa(i),0,0}
	}
	m := Master{
		timeout : 	10*time.Second,
		reduceN:	nReduce,
		mapN:		len(files),
		mapWork:	mapWork,
		reduceTable:reduceTable,
		allFinished:false,
	}


	// 得到所有原始slide 文件和要确定的reduce数量
	//分配worker：map去查读取这些文件

	m.server()
	return &m
}

```

worker

```go
package mr

import "fmt"
import "log"
import "net/rpc"
import "hash/fnv"
import "io/ioutil"
import "encoding/json"
import "os"
import "strconv"
import "sort"
//import "time"

//
// Map functions return a slice of KeyValue.
//
type KeyValue struct {
	Key   string
	Value string
}
type ByKey []KeyValue

// for sorting by key.
func (a ByKey) Len() int           { return len(a) }
func (a ByKey) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
func (a ByKey) Less(i, j int) bool { return a[i].Key < a[j].Key }

//
// use ihash(key) % NReduce to choose the reduce
// task number for each KeyValue emitted by Map.
//
func ihash(key string) int {
	h := fnv.New32a()
	h.Write([]byte(key))
	return int(h.Sum32() & 0x7fffffff)
}

//
// main/mrworker.go calls this function.
//
func Worker(mapf func(string, string) []KeyValue,
	reducef func(string, []string) string) {
	//接收一个map函数和一个reduce函数
	//调用一个call的包装方法里面定义好arg和reply
	for{
		response:=CallExample()
		if response.AllFinished{
			break
		}
		if response.Type==1{
			MapImp(response,mapf)
			if CallFinish(response.Id,response.Type).AllFinished{
				return
			}
		}else if response.Type==2{
			ReduceImp(response,reducef)
			if CallFinish(response.Id,response.Type).AllFinished{
				return
			}
		}
		//time.Sleep(time.Second)

	}
}

func MapImp(response MyRPCReplay,mapf func(string, string) []KeyValue){
	//分配map
	//读取文件
	file, err := os.Open(response.Filename)
	if err != nil {
		log.Fatalf("cannot open %v", response.Filename)
	}
	content, err := ioutil.ReadAll(file)
	if err != nil {
		log.Fatalf("cannot read %v", response.Filename)
	}
	file.Close()
	kva := mapf(response.Filename, string(content))
	//分配kva 写到 rm-X-Y file 中
	reduceN:=response.ReduceN
	//初始化数组
	splitKey := make([][]KeyValue,reduceN)
	for i:=range splitKey{
		splitKey[i]=[]KeyValue{}
	}
	//把kva分配进每个splitKey[i]
	for _,kv:=range kva{
		y:=ihash(kv.Key)%response.ReduceN
		splitKey[y]=append(splitKey[y],kv)
	}
	for i:=range splitKey{
		if IsExist("mr-"+strconv.Itoa(response.Id)+"-"+strconv.Itoa(i)){
			fmt.Println("mr-"+strconv.Itoa(response.Id)+"-"+strconv.Itoa(i)+"文件存在了呀")
		}
		Wfile, _ :=os.Create("mr-"+strconv.Itoa(response.Id)+"-"+strconv.Itoa(i))
		enc := json.NewEncoder(Wfile)
		for _,kv := range splitKey[i]{
			err:=enc.Encode(kv)
			if err!=nil{
				break
			}
		}
		Wfile.Close()
	}
}

func ReduceImp(response MyRPCReplay,reducef func(string, []string) string){
	//创建filename

	id:=response.Id
	mapNum:=response.MapN
	kva := []KeyValue{}
	for i:=0;i<mapNum;i++{
		filename:="mr-"+strconv.Itoa(i)+"-"+strconv.Itoa(id)
		file,err:=os.Open(filename)
		if err!=nil{
			fmt.Println(err)
			break
		}
		dec := json.NewDecoder(file)
		for{
			var kv KeyValue
			if err := dec.Decode(&kv);err!=nil{
				break
			}
			kva = append(kva,kv)
		}

	}

	sort.Sort(ByKey(kva))
	i := 0
	if IsExist(response.Filename){
		fmt.Println("文件存在了"+response.Filename)
	}
	ofile, _ := os.Create(response.Filename)
	for i < len(kva) {
		j := i + 1
		for j < len(kva) && kva[j].Key == kva[i].Key {
			j++
		}
		values := []string{}
		for k := i; k < j; k++ {
			values = append(values, kva[k].Value)
		}

		output := reducef(kva[i].Key, values)

		fmt.Fprintf(ofile, "%v %v\n", kva[i].Key, output)

		i = j
	}
	ofile.Close()
	//等待全部完成进行删除
	for i:=0;i<mapNum;i++{
		filename:="mr-"+strconv.Itoa(i)+"-"+strconv.Itoa(id)
		os.Remove(filename)
	}
}
//
// example function to show how to make an RPC call to the master.
//
// the RPC argument and reply types are defined in rpc.go.
//
func CallExample() MyRPCReplay{

	// declare an argument structure.
	args := MyRPCArgs{}

	// fill in the argument(s).
	args.Status = "apply"

	// declare a reply structure.
	reply := MyRPCReplay{}
	// send the RPC request, wait for the reply.
	if !call("Master.Handler", &args, &reply){
		//拨通 并且返回了 reply
		reply.Type=0
		return reply
	}
	//fmt.Println(reply)
	// reply.Y should be 100.
	//fmt.Printf("reply.Y %v %v\n", reply.allFinished,reply.Id)

	return reply
}
func CallFinish(id int,Type int)MyRPCReplay{
	// declare an argument structure.
	args := MyRPCArgs{}

	// fill in the argument(s).
	args.Status = "commit"
	args.Id=id
	args.Type=Type

	// declare a reply structure.
	reply := MyRPCReplay{}

	// send the RPC request, wait for the reply.
	if !call("Master.Handler", &args, &reply){
		//拨通 并且返回了 reply
		reply.Type=0
		return reply
	}



	// reply.Y should be 100.
	return reply
}
//
// send an RPC request to the master, wait for the response.
// usually returns true.
// returns false if something goes wrong.
//
func call(rpcname string, args interface{}, reply interface{}) bool {
	// c, err := rpc.DialHTTP("tcp", "127.0.0.1"+":1234")
	sockname := masterSock()
	c, err := rpc.DialHTTP("unix", sockname)
	if err != nil {
		log.Fatal("dialing:", err)
		return false
	}
	defer c.Close()
	//rpc调用master中的方法rpcname为要调用的方法名
	err = c.Call(rpcname, args, reply)
	if err == nil {
		return true
	}

	//fmt.Println(err)
	return false
}
func IsExist(path string) bool {
	_, err := os.Stat(path)
	return err == nil || os.IsExist(err)
	// 或者
	//return err == nil || !os.IsNotExist(err)
	// 或者
	//return !os.IsNotExist(err)
}
```

rpc

```go
package mr

//
// RPC definitions.
//
// remember to capitalize all names.
//

import "os"
import "strconv"

//
// example to show how to declare the arguments
// and reply for an RPC.
//

// Add your RPC definitions here.
type MyRPCArgs struct {
	Status string// apply commit
	Type int// 1 map 2 reduce
	Id int

}
type MyRPCReplay struct {
	Type     int
	//Type 1为map
	//Type 2为reduce
	Filename string
	ReduceN int
	MapN int
	Id int
	AllFinished bool
}

// Cook up a unique-ish UNIX-domain socket name
// in /var/tmp, for the master.
// Can't use the current directory since
// Athena AFS doesn't support UNIX-domain sockets.
func masterSock() string {
	s := "/var/tmp/824-mr-"
	s += strconv.Itoa(os.Getuid())
	return s
}

```

毕竟第一次写 go 语言，感觉写出来的东西特别乱。

## 收获

收获肯定是满满的，为了写 lab 学了 go 语言，了解到了这个 go 的协程和 python 的协程非常不同。再来就是等于这下初入门了分布式了，踏出第一步相信后面会越来越顺，毕竟万事开头难。最后就是花了大量的时间在看 sh 测试脚本，虽然没有系统的学，但下次看脚本也会不慌了。

然后下一步就是 lab2 。加油！