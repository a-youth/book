

开放一个免费 IP 定位 (GEO IP) 接口



最近前端有个IP定位的需求，经过1周的折腾搞定。公司就是用这套代码，已上线一周，正常运行。随后在博客上部署了一套API接口，免费开放出来。

如何获取token，查看完整文档 请移步：

[免费ip地理位置(GEO IP)接口](https://www.douyacun.com/article/a57b58a343f051cf1fb9761a31d37693)

**特点：**

1. 支持传参IP定位，如果不传参数IP，默认使用当前客户端的IP地址定位
2. 支持前端jsonp调用，避免跨域

**限流**

- QPS 默认 5/s, 最大并发10/s, 10万次/天， 
- QPS 可以增加到 10/s，最大并发20/s, 100万次/天，够用了。。。。
- 更大频次需要支付流量费用，100元长期支持，不限频次

