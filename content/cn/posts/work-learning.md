---
title: "知识补充区"
date: 2022-07-01T22:07:24+08:00
draft: false
categories: ['work']
keywords: ['work']
description: "工作知识补充区"
---

{{< music id="27588968" >}}


### Git
* [GitHub中origin和upstream的区别](https://blog.51cto.com/u_15127588/4728372)
* [Git commit 使用及规范](https://www.jianshu.com/p/ff4f98695c2c)
* [How to convert `git:` urls to `http:` urls](https://stackoverflow.com/questions/1722807/how-to-convert-git-urls-to-http-urls)
* [Git通过配置config维护多个账号，区分个人账号和公司账号](https://blog.csdn.net/CoderBruis/article/details/120674608)
* [git commit](https://github.com/Zhengqbbb/cz-git)
* [Gitmoji](https://github.com/carloscuesta/gitmoji)
* [GitHook](https://razeen.me/posts/golang-and-git-commit-message-pre-commit/)


### Golang
* [Go的构建约束](https://polarisxu.studygolang.com/posts/go/dynamic/go1.17-build-contraints/)
* [Go Plugin](https://tonybai.com/2021/07/19/understand-go-plugin/)
* [Go基础0x02-go build -tags使用](https://www.cnblogs.com/JasonCeng/p/15134495.html)
* [Gob序列化](https://www.cnblogs.com/yinzhengjie2020/p/12735277.html)
* [ants协程池](https://github.com/panjf2000/ants)
* [golang在编译时用ldflags设置变量的值](https://studygolang.com/articles/9422)
* [dlv调试](https://chai2010.cn/advanced-go-programming-book/ch3-asm/ch3-09-debug.html)
* [Go Generic](https://segmentfault.com/a/1190000041634906)
* [Go-mod依赖及可视化](http://www.zengyuzhao.com/archives/348)
* [Golang 的骚操作：go:linkname](https://zhuanlan.zhihu.com/p/440622708)
* [反序列化json默认类型](https://pkg.go.dev/encoding/json#Decoder.Decode)

```
bool, for JSON booleans
float64, for JSON numbers
string, for JSON strings
[]interface{}, for JSON arrays
map[string]interface{}, for JSON objects
nil for JSON null
```

* 正则替换

```
func replace(text string, replacement map[string]string) string {
	var leftPad, rightPad = `{{`, `}}`
	var rgx = regexp.MustCompile(leftPad + "(.*?)" + rightPad)
	rs := rgx.ReplaceAllFunc([]byte(text), func(bytes []byte) []byte {
		key := bytes[len(leftPad) : len(bytes)-len(rightPad)]
		log.Println(string(key))
		if val, ok := replacement[string(key)]; ok && val != "" {
			return []byte(val)
		}
		return bytes
	})
	return string(rs)
}
```

### Grafana
* [grafana-tutorial](https://github.com/Kalasearch/grafana-tutorial)
* [Prometheus+Grafana学习]()
* [Go进阶31:Prometheus Client教程](https://mojotv.cn/go/prometheus-client-for-go)
* [Google mtail配合Prometheus和Grafana实现自定义日志监控](https://segmentfault.com/a/1190000040503959)
* [docker compose](https://github.com/retzkek/chiamon)
* [使用loki+ mtail + grafana + prometheus server分析应用问题](https://www.cnblogs.com/rongfengliang/p/10117107.html)

### Ansible
* [Ansible Playbook 入门指南](https://segmentfault.com/a/1190000020523508)
* [Playbooks 介绍](https://ansible-tran.readthedocs.io/en/latest/docs/playbooks_intro.html)

### Draw
* [使用 dot 画图工具](https://jeanhwea.github.io/article/drawing-graphs-with-dot.html#org8e4fe4c)

### Goland
* [器 | Goland 设置注释模板](https://juejin.cn/post/6982554895987507230)
* [Swagger](https://www.lixueduan.com/post/go/swagger/)

### Linux
* [iptables简单使用例子](https://emacsist.github.io/2016/10/09/iptables%E7%AE%80%E5%8D%95%E4%BD%BF%E7%94%A8%E4%BE%8B%E5%AD%90/)
* [Makefile 小记](https://howiezhao.github.io/2019/04/18/makefile/)

### Asciinema
* [asciinema](https://asciinema.org/)
* [asciinema to svg](https://github.com/marionebl/svg-term-cli)

### Jaeger
* [开源分布式追踪系统 — Jaeger介绍](http://t.zoukankan.com/ChangAn223-p-11458226.html)

### SSH
* [隧道代理](https://kanda.me/2019/07/01/ssh-over-http-or-socks/)

### Service Mesh
* [什么是 Service Mesh](https://zhuanlan.zhihu.com/p/61901608)
* [我的ServiceMesh学习之旅](https://bbs.huaweicloud.com/blogs/345339)
* [(2017)Pattern: Service Mesh](https://skyao.io/learning-servicemesh/docs/introduction/recommended/pattern_service_mesh.html)

### Python
* [How to capture HTTP requests using Selenium](https://www.dilatoit.com/2020/12/17/how-to-capture-http-requests-using-selenium.html)


### 压测方案
* [Performance test](https://github.com/panjianning/performance-test)

### Go Useful Tools
* [golang.org/x/sync/singleflight](golang.org/x/sync/singleflight)，防止缓存击穿
* [golang.org/x/sync/errgroup](golang.org/x/sync/errgroup)
* [github.com/sony/gobreaker](github.com/sony/gobreaker)，断路器
* [github.com/meirf/gopart](github.com/meirf/gopart)，方便并发切片
* [golang.org/x/time/rate](golang.org/x/time/rate)，本地限流，可日志限流打印
* [golang.org/x/sync/semaphore](golang.org/x/sync/semaphore)