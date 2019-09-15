Redis的发布订阅功能由PUBLISH, SUBSCRIBE, PSUBSCRIBE等命令组
成.
1. 频道的订阅与退订
客户端执行SUBSCRIBE命令订阅某个或某些频道. 
举个例子, 假设A, B两个客户端都执行SUBSCRIBE命令:
如果另外一个客户端执行 publish "news.it" "hello world"命令, 向 
news.it发送消息 "hello world", 那么 "news.it" 的两个订阅者都可
以收到这个消息.
(1)Redis将所有频道的订阅关系都保存在了服务器装的
pubsub_channels字典里面, 这个字典的键是某个被订阅的频道, 而键
的值则是一个链表. 链表中记录了所有订阅这个频道的客户端.
2. 模式的订阅和退订
服务器将所有模式的订阅关系都保存在服务器状态里面
3. 