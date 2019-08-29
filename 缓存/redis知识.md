[TOC]
# 实用链接
https://zhuanlan.zhihu.com/p/33054749
https://cloud.tencent.com/developer/article/1148772
https://gblog.sherlocky.com/redis/
[redis开发规范](https://www.jianshu.com/p/a04be12fe8fc)
[网易的redis-ncr](https://sq.163yun.com/blog/article/183664506190012416)
# 内存突增可能原因
1. bigkey：命令redis-cli --bigkeys -i 0.1 -h 127.0.0.1查询bigkey
2. 键值对个数增加
3. 客户端缓冲区：命令 info clients，观察是否明显的omem大于0的情况；
4. redis的kv哈希表做了rehash

# redis LFU热点key发现
[redis lfu原理解析](https://yq.aliyun.com/articles/278922?utm_content=m_40440)

