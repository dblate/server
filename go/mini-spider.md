## 题目
```
[背景]:
    在调研过程中，经常需要对一些网站进行定向抓取。由于go包含各种强大的库，使用go做定向抓取比较简单。请使用go开发一个迷你定向抓取器mini_spider，实现对种子链接的抓取，并把URL长相符合特定pattern的网页保存到磁盘上。

[程序运行]：
   ./mini_spider -c ../conf -l ../log

[配置文件spider.conf]：
    [spider]
    # 种子文件路径
    urlListFile = ../data/url.data
    # 抓取结果存储目录
    outputDirectory = ../output
    # 最大抓取深度(种子为0级)
    maxDepth = 1
    # 抓取间隔. 单位: 秒
    crawlInterval =  1
    # 抓取超时. 单位: 秒
    crawlTimeout = 1
    # 需要存储的目标网页URL pattern(正则表达式)
    targetUrl = .*.(htm|html)$
    # 抓取routine数
    threadCount = 8

[种子文件为json格式，示例如下]：
   [
     "http://www.baidu.com",
     "http://www.sina.com.cn",
     ...
   ]

[需要的能力]：
* 抓取网页 √
* 读取命令行参数 √
* 读取配置文件 √
* 读取文件内容（File => String）√
* 写日志 √
* 解析HTML √
* 多线程并行

[要求和注意事项]：
    1. 需要支持命令行参数处理。具体包含:  -h(帮助)、-v(版本)、-c(配置文件路径）、-l（日志文件路径，2个日志：mini_spider.log和mini_spider.wf.log)
    2. 抓取网页的顺序没有限制
    3. 单个网页抓取或解析失败，不能导致整个程序退出。需要在日志中记录下错误原因并继续。
    4. 当程序完成所有抓取任务后，必须优雅退出
    5. 从HTML提取链接时需要处理相对路径和绝对路径
    6. 需要能够处理不同字符编码的网页(例如utf-8或gbk)
    7. 网页存储时每个网页单独存为一个文件，以URL为文件名。注意对URL中的特殊字符，需要做转义
    8. 要求支持多routine并行抓取（注意：这里并不是指简单设置GOMAXPROCS>1)
    9. 代码严格遵守百度Golang编码规范V1.1（http://wiki.baidu.com/pages/viewpage.action?pageId=104882818）
    10. 不仅仅考察编程规范，同时也考察编程思路：代码的可读性和可维护性要好，合理进行pakcage、函数的设计和划分，多线程间的同步、信号量的使用等
    11. 完成相应的单元测试和使用demo。你的demo必须可运行，单元测试有效而且通过
    12. 抓取要实现“去重”功能，即对于已经抓取过的网页不要重复抓取
    13. 选做功能：控制抓取间隔，避免对方网站封禁百度IP
       - 设计上可以针对单个routine控制，也可以针对站点进行控制
       - 注：这个功能的实现有一定难度，有余力的同学可以尝试
    14. 在实现中，可以使用以下外部库：
         - 日志库请使用http://icode.baidu.com/repos/baidu/go-lib/log
         - 主配置文件读取使用https://github.com/go-gcfg/gcfg
         - html解析请使用 https://go.googlesource.com/net/
         - 队列可使用http://icode.baidu.com/repos/baidu/go-lib/queue
         - 去重可使用系统库sync.Map
    15. 编译环境配置请参考：http://wiki.baidu.com/pages/viewpage.action?pageId=917601678
    16. 对于配置文件的异常，需要进行检查和处理
```

## 调研

### 网页爬取
```golang
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
	"os"
	"regexp"
	"time"
)

var RE = regexp.MustCompile("\\w+\\.\\w+$")

func main() {
	urls := []string{
		"http://www.qq.com",
		"http://www.163.com",
		"http://www.sina.com",
	}

	start := time.Now()

	for _, url := range urls {
		start := time.Now()
		res, err := http.Get(url)
		if err != nil {
			fmt.Fprintf(os.Stderr, "fetch: %v\n", err)
			os.Exit(1)
		}

		body, err := ioutil.ReadAll(res.Body)
		res.Body.Close()

		if err != nil {
			fmt.Fprintf(os.Stderr, "fetch: reading %s: %v\n", url, err)
			os.Exit(1)
		}

		fileName := getFileName(url)

		ioutil.WriteFile(fileName, body, 0644)
		elapsed := time.Since(start).Seconds()

		fmt.Printf("%.2fs %s\n", elapsed, fileName)
	}

	elapsed := time.Since(start).Seconds()
	fmt.Printf("%.2fs elapsed\n", elapsed)
}

func getFileName(url string) string {
	return RE.FindString(url) + ".txt"
}
```

### html解析
```golang
package main

import (
	"fmt"
	"log"
	"strings"

	"golang.org/x/net/html"
)

func main() {
	s := `<p>Links:</p><ul><li><a href="goo">Goo</li><a href="/bar/baz">BarBaz</ul><div><a href="another">another</div>`
	doc, err := html.Parse(strings.NewReader(s))
	if err != nil {
		log.Fatal(err)
	}
	var f func(*html.Node)
	f = func(n *html.Node) {
		if n.Type == html.ElementNode && n.Data == "a" {
			for _, a := range n.Attr {
				if a.Key == "href" {
					fmt.Println(a.Val)
					break
				}
			}
		}
		for c := n.FirstChild; c != nil; c = c.NextSibling {
			f(c)
		}
	}
	f(doc)
}
```

### 配置文件解析
#### 文件内容读取(File => String)
```golang
package main

import (
	"fmt"
	"io/ioutil"
	"log"
)

func main() {
	bytes, err := ioutil.ReadFile("./spider.conf")
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println("Bytes read: ", len(bytes))
	fmt.Println("String: ", string(bytes))
}
```

#### 配置文件内容解析
```golang
package main

import (
	"fmt"
	"io/ioutil"
	"log"

	"gopkg.in/gcfg.v1"
)

func main() {
	// cfgStr := `
	// [spider]

	// # 种子文件路径

	// urlListFile = ../data/url.data

	// # 抓取结果存储目录

	// outputDirectory = ../output

	// # 最大抓取深度(种子为0级)

	// maxDepth = 1

	// # 抓取间隔. 单位: 秒

	// crawlInterval =  1

	// # 抓取超时. 单位: 秒

	// crawlTimeout = 1

	// # 需要存储的目标网页URL pattern(正则表达式)

	// targetUrl = .*.(htm|html)$

	// # 抓取routine数

	// threadCount = 8
	// `

	bytes, readErr := ioutil.ReadFile("./spider.conf")
	if readErr != nil {
		log.Fatal(readErr)
	}
	cfgStr := string(bytes)

	cfg := struct {
		Spider struct {
			Name            string
			UrlListFile     string
			OutputDirectory string
			MaxDepth        int
			CrawlInterval   int
			CrawlTimeout    int
			TargetUrl       string
			ThreadCount     int
		}
	}{}

	err := gcfg.ReadStringInto(&cfg, cfgStr)
	if err != nil {
		log.Fatalf("Failed to parse gcfg data: %s", err)
	}
	fmt.Println(cfg.Spider.UrlListFile)
	fmt.Println(cfg.Spider.TargetUrl)
}
```
### 打印日志
```golang
package main

import (
	"time"

	"icode.baidu.com/baidu/go-lib/log"
)

func main() {
	log.Init("test", "INFO", "./log", true, "midnight", 5)
	log.Logger.Warn("warn msg")
	log.Logger.Info("info msg")

	time.Sleep(100 * time.Millisecond)
}
```

### 命令行参数获取
```golang
package main

import (
	"flag"
	"fmt"
)

var confFlag string

func init() {
	flag.StringVar(&confFlag, "c", "default conf path", "input relative path of conf file")
}

func main() {
	var nFlag = flag.Int("n", 1234, "help message for flag n")
	flag.Parse()
	fmt.Printf("nFlag: %v\n", *nFlag)
	fmt.Printf("confFlag: %v\n", confFlag)
}
```
