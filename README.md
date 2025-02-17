# linux_dns_process
linux平台监听发起DNS请求的进程，打印相关的**进程树**，支持指定监听的域名。

go语言+cilium(bpf2go)框架+ebpf技术


## 工具特点
+ go语言开发，直接打包成ELF可执行文件，仅需目标环境支持BTF，无需安装BCC等编译相关依赖
+ hook的udp_send_skb()函数
+ 支持指定监听的域名，仅输出请求指定域名的进程
+ 直接打印进程树，方便定位
+ **针对当前运行系统中的内核函数名称，动态hook(详情请参考“实现细节”部分)**
+ 从/proc目录下获取的进程信息



## 支持的系统内核版本

系统的内核版本最好 >= 5.2 (内核需要启用 BPF 和 BTF 支持）

对于内核版本介于 4.18 ~ 5.2 之间的系统，如果系统中未提供程序依赖的内核 BTF 文件的话，
将自动尝试从 [龙蜥 BTF 目录](https://mirrors.openanolis.cn/coolbpf/btf/) 和 [BTFhub](https://github.com/aquasecurity/btfhub-archive) 
下载当前系统内核版本对应的 BTF 文件
> 此部分代码直接copy的ptcpdump项目的, https://github.com/mozillazg/ptcpdump


## 效果演示
（一）不指定域名

<img width="753" alt="image" src="https://github.com/user-attachments/assets/51421c43-4f91-499a-9655-005cc1a16015" />

（二）指定域名，仅输出请求指定域名的进程

<img width="747" alt="image" src="https://github.com/user-attachments/assets/bbb766b2-c934-496d-8102-19df51730135" />



## 实现细节
### 1. 为什么hook udp_send_skb()函数
linux内核向用户态提供统一的函数入口，而越往下则会按不同协议拆分进行不同函数的调用。这里主要关注udp_sendmsg()和udp_send_skb()两个函数。
由于udp协议既可以每次sendto指定目的地址，也支持先connect绑定目的地址再发送。

两种方式的差异导致目的IP+端口信息在udp_sendmsg()函数不同位置的参数中

为了程序编写方便，我们选择另一个获取信息相对统一的函数udp_send_skb()进行hook。
> 参考：https://www.ctfiot.com/41504.html


### 2. 相比ptcpdump有何优势
(1) 程序会先通过`cat /proc/kallsyms | grep udp_send_skb`获取需要hook的内核函数名称

ptcpdump hook的是固定的udp_send_skb，**但是在有的环境中该内核函数并不是这个名称，导致抓不到包**


如下图所示，在这个环境中，hook的应该是 `udp_send_skb.isra.51`


<img width="554" alt="image" src="https://github.com/user-attachments/assets/2a7badfb-12fa-4d74-881d-cf155b01eca3" />



`udp_send_skb.isra.51` 这种符号命名通常出现在 Linux 内核或编译器优化后的函数符号中，isra 代表 interprocedural scalar replacement of aggregates（跨过程标量替换聚合），而 51 是优化过程中生成的标识符

具体解释
1.	isra 优化
  + isra 是 GCC 编译器 进行 跨过程优化（IPO，Interprocedural Optimization） 时的标志
  + ISRA（Interprocedural Scalar Replacement of Aggregates） 是 GCC 的一种优化技术，它会将结构体或数组等聚合类型（aggregates）拆解成独立的标量变量，以减少不必要的内存访问，提升 CPU 指令效率
2.	51 是编译器生成的唯一 ID
  + 这个数字可能是 GCC 编译时生成的唯一标识符，表示这是 该优化过程中生成的特定版本，不同编译参数可能会导致不同的编号




(2) 支持直接输出进程树

(3) 无需任何参数，简单易用


## 待完成
+ 使用BTFGen直接打包精简的BTF文件，不需要再运行时下载


## 参考
https://github.com/mozillazg/ptcpdump

