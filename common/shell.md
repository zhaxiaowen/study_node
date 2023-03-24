# shell 脚本练习

## [awk 分析 nginx 日志](https://blog.csdn.net/weixin_36344231/article/details/116753865)

* 统计日志中访问最多的 10 个 Ip	

```
awk '{a[$1]++}END{for(i in a)print a[i],i}|"sort -k1 -nr|head -n10"' access.log

a[$1]++ 创建数组a，以第一列作为下标，使用运算符++作为数组元素，元素初始值为0。处理一个IP时，下标是IP，元素加1，处理第二个IP时，下标是IP，元素加1，如果这个IP已经存在，则元素再加1，也就是这个IP出现了两次，元素结果是2，以此类推。因此可以实现去重，统计出现次数。

-k1:指定按第一列排序
-nr: -n是按数字排序,不指定n,默认将数字按字符排序 -r:倒序
```

```
awk '{print $1} access.log|sort|uniq -c|sort -k1 -nr|head -n10'
```

* 统计日志中访问大于 100 次的 ip

  ```
  awk '{a[$1]++}END{for(i in a){if(a[i]>100)print a[i],i}}' access.log
  ```
