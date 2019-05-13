扩展模块：
- `iprange` 用来设置连续的 `ip` 范围匹配条件, 例如：
    ```shell
    iptables -t filter -I INPUT -m iprange --src-range 192.168.1.127-192.168.1.146 -j DROP
    iptables -t filter -I OUTPUT -m iprange --dst-range 192.168.1.127-192.168.1.146 -j DROP
    iptables -t filter -I INPUT -m iprange ! --src-range 192.168.1.127-192.168.1.146 -j DROP
    ```

- `string` 设置数据包内容的字符串匹配条件，例如：
    ```shell
    #示例
    iptables -t filter -I INPUT -p tcp --sport 80 -m string --algo bm --string "OOXX" -j REJECT
    iptables -t filter -I INPUT -p tcp --sport 80 -m string --algo bm --string "OOXX" -j REJECT
    ```
    > 其中 --algo只是字符串匹配算法（bm 好后缀坏字符算法）
- `time` 设置数据包传输的时间规则，例如：
    ```shell
    #示例
    iptables -t filter -I OUTPUT -p tcp --dport 80 -m time --timestart 09:00:00 --timestop 19:00:00 -j REJECT
    iptables -t filter -I OUTPUT -p tcp --dport 443 -m time --timestart 09:00:00 --timestop 19:00:00 -j REJECT
    iptables -t filter -I OUTPUT -p tcp --dport 80  -m time --weekdays 6,7 -j REJECT
    iptables -t filter -I OUTPUT -p tcp --dport 80  -m time --monthdays 22,23 -j REJECT
    iptables -t filter -I OUTPUT -p tcp --dport 80  -m time ! --monthdays 22,23 -j REJECT
    iptables -t filter -I OUTPUT -p tcp --dport 80  -m time --timestart 09:00:00 --timestop 18:00:00 --weekdays 6,7 -j REJECT
    iptables -t filter -I OUTPUT -p tcp --dport 80  -m time --weekdays 5 --monthdays 22,23,24,25,26,27,28 -j REJECT
    iptables -t filter -I OUTPUT -p tcp --dport 80  -m time --datestart 2017-12-24 --datestop 2017-12-27 -j REJECT
    ```
    > 可以以周和月为周期设置，也可以设置时间或日期范围，其中以周和月设置是，支持取反符号"!"

- `connlimit` 设置网络连接数规则，可以选择连接数是航线，以及连接的ip 掩码，例如：
    ```shell
    #示例
    iptables -I INPUT -p tcp --dport 22 -m connlimit --connlimit-above 2 -j REJECT
    iptables -I INPUT -p tcp --dport 22 -m connlimit --connlimit-above 20 --connlimit-mask 24 -j REJECT
    iptables -I INPUT -p tcp --dport 22 -m connlimit --connlimit-above 10 --connlimit-mask 27 -j REJECT
    ```
- `limit` 设置**保温到达速率**的规则，可以以 second、minute、hour、day为单位进行设置，例如：
    ```bash
    #示例 #注意，如下两条规则需配合使用，具体原因上文已经解释过，忘记了可以回顾。
    iptables -t filter -I INPUT -p icmp -m limit --limit-burst 3 --limit 10/minute -j ACCEPT 
    iptables -t filter -A INPUT -p icmp -j REJECT 
    ```
    > 这里有令牌桶的概念，limit设置包数，limit-burst设置令牌桶的容量
    

> 参考至：http://www.zsythink.net/archives/1564
