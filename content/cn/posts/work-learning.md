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


### Golang
* [Go的构建约束](https://polarisxu.studygolang.com/posts/go/dynamic/go1.17-build-contraints/)
* [Go Plugin](https://tonybai.com/2021/07/19/understand-go-plugin/)
* [Go基础0x02-go build -tags使用](https://www.cnblogs.com/JasonCeng/p/15134495.html)
* [Gob序列化](https://www.cnblogs.com/yinzhengjie2020/p/12735277.html)
* [ants协程池](https://github.com/panjf2000/ants)
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