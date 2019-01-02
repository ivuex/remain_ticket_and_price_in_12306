<!--
.. title: 爬取12306余票信息和票价
.. slug: pa-qu-12306yu-piao-xin-xi-he-piao-jie
.. date: 2018-12-23 01:27:01 UTC+08:00
.. tags: requests sqlite3 prettytalbe colorama
.. category: requests sqlite3 prettytalbe colorama
.. link: 
.. description: 
.. type: text
-->

## 分析
<!--12306车票的信息抓取来还是比较简单的，
其实难的如何用Python抢票，12306有一个很“特殊”的验证码，即使是人识别起来都有一定难度。
但其实这种难度也恰是它薄弱的地方，这意味着此类验证码的数量是有限的。
我自己没有抓过，但我之前看过一篇文章，据说这种码一共是9000个左右（我记得好像作者把验证码抓下来用md5加密后保存下来，存到9000多个就存不下去了）。
假设我们把每个验证码图片的答案都记下来，理论上就可以完成自动登陆了。听起来操作很费力。-->
<!--如果只是要查询车票信息和价钱的话，就比较简单了。只有几个需要注意的点，这里讲一下。-->

#### 获得站名和其编号的对应关系
 + 所有请求中，有一个js文件很重要，名字叫station_name.js?station_version=1.9053,后面的是版本号
 + 这个文件会返回一个字符串，类似于： @bjb|北京北|VAP|beijingbei|bjb|0@bjd|北京东|BOP|beijingdong|bjd|1.......... 可以得出以下信息
    - 站与站之间用@分隔
    - 然后每个站的信息字段用|分隔
    - 其中最重要的是第三个字段，之后查询车票信息的时候，就是用这个字段的值的
    <!--我不仅获取并解析了这个字符串，同时也把信息写入了sqlite3数据库里。-->

#### 查询剩余车票信息
 + 查询余票信息的是一条XHR请求：
```
https://kyfw.12306.cn/otn/leftTicket/queryX?leftTicketDTO.train_date=2018-05-24&leftTicketDTO.from_station=BJP&leftTicketDTO.to_station=TJP&purpose_codes=ADULT
```
 + 其中train_data和purpose_codes很好理解
 + 两个station用的就是上一节中所说的teleCode，因此要模拟这个请求，你需要先根据站点名字（如‘北京北’）从数据库中得到其编号（如‘VAP'）。

###### 这条请求的响应中，有三部分有用：
 + status字段，标识了这条请求是否成功。
 + data字段中的map字段，里面包好了响应中使用的tecoCode与其对应的站点名字，这样我们就不需要再利用数据库将编号转成站台名了
 + 因为采用的模糊查询，所以map中数据可能不止两个，比如说搜索’北京‘可能会返回’北京‘和’北京南‘
 + data字段中的result字段，这个字段就包含了查询票价需要的信息。但是它也是经过编码的同样用|分隔
    - 比如"null|预订|240000D3110K|D311|VNP|SHH|VNP|TXP|21:11|22:19|01:08|N|0Xz%2FAQnjcOjRAf%2FlTLDixMnMJwxcpe7x|20180524|3|P2|01|02|0|0||||无||||||||||无|F040|F4|0"
 <!--+ 我们可以从网页源代码和js文件里找出一点线索。在queryLeftTicket_end_js.js?scriptVersion=1.9085文件中的2814行，可以看到:-->



<!--这已经可以看出很多东西了，如果还有点迷惑，再看网页的源代码：-->



<!--通过审查元素，你可以知道哪个id对应哪个字段，然后再从上图上找出其对应的字段偏移量。-->

#### 查询票价
 + 查询票价使用的是另外一条XHR请求：
```
https://kyfw.12306.cn/otn/leftTicket/queryTicketPrice?train_no=24000C22290F&from_station_no=01&to_station_no=03&seat_types=O9OMP&train_date=2018-05-24
```
<!--这几个参数在查询余票的响应中都包含了，分别是第2、16、17和35个字段。-->
<!--但是难点是找出seat_types中几个值分别代表了什么座位。-->
<!--这个我没找到线索，但是我在网上找到了别人代码里写的对应关系，但是现在找不到那份代码了。-->
 + 以下是seat_types中的值与座位的对应关系

|座位类型|编号|
|:----|:-:|
|商务座|A9|
|特等座|P|
|商务座|A9|
|特等座|P|
|一等座|M|
|二等座|O|
|高级软卧	|A6|
|软卧|A4|
|动卧|--|
|硬卧|A3|
|软座|A2|
|硬座|A1|
|无座|WZ|
|其他|--|

 + 一条余票查询的请求会对应很多条票价查询的请求，因此如果余票类型很充足，会消耗很长时间（可以关闭）。

## 编码

###### 实现的功能
 + 余票数量
 + 票价
 + 使用了PrettyTable和colorama库，支持彩色表格打印
 + 增加是否支持身份证出入站和始发站、终点站、经过站的标识

<!--代码如下，不包括数据库接口和配置文件，其中t12306_init()只要执行一次（省资源，执行多次不会出错）：-->
####### 代码
```
# -*- coding:utf-8 -*-
# filename: 12306.py

import requests
import ast
import json
from random import choice
from prettytable import PrettyTable
from colorama import init, Fore, Style


import Configure as Configs
import Sqlite3api as Sqlite3

init()

header = {}
header['user-agent'] = choice(Configs.FakeUserAgents)
header['Referer'] = "https://kyfw.12306.cn/otn/leftTicket/init"

def t12306_init():
    conn = Sqlite3.sqlite3_init()

    url = "https://kyfw.12306.cn/otn/resources/js/framework/station_name.js?station_version=1.9053"
    content = None

    ret = Sqlite3.sqlite3_execute(conn, "SELECT count(*) FROM sqlite_master WHERE type='table' AND name='t12306'")[0][0]
    if ret == 1:
        Sqlite3.sqlite3_execute(conn, "DROP TABLE t12306")

    Sqlite3.sqlite3_execute(conn, "CREATE TABLE t12306 (stationId real, stationName text, teleCode text, pinYin text, pinYinHead text)")
    Sqlite3.sqlite3_execute(conn, "CREATE UNIQUE INDEX idx_follow_stationId on t12306(stationId)")

    try:
        response = requests.get(url, headers=header)

        if response.status_code == requests.codes.ok:
            content = response.text
    except Exception as e:
        print (e)

    data = content[:-1].split('=')[1][2:]

    for station in data.split('@'):
        fields = station.split('|')
        Sqlite3.sqlite3_execute(conn, "INSERT INTO t12306 VALUES (?,?,?,?,?)", (fields[5],fields[1],fields[2],fields[3],fields[0],))
        #print (fields[5],fields[1],fields[2],fields[3],fields[0])

    Sqlite3.sqlite3_close(conn)


def check_left_ticket(train_date, from_station, to_station, purpose_codes, need_price=False):
    url = "https://kyfw.12306.cn/otn/leftTicket/query"

    payload = {
        'leftTicketDTO.train_date': train_date,
        'leftTicketDTO.from_station': from_station,
        'leftTicketDTO.to_station': to_station,
        'purpose_codes': purpose_codes
    }

    content = ''
    try:
        response = requests.get(url, headers=header, params=payload)
        if response.status_code == requests.codes.ok:
            response.encoding = 'utf-8'
            content = response.text
    except Exception as e:
        print (e)

    data = json.loads(content)
    #print (data)
    if data.get('status') == False:
        print ("获取数据失败。")
        return

    # 站点编号->站点名字 的Map
    name_map = data.get('data').get('map')

    table = PrettyTable()
    table.field_names = ["车次", "出发/到达", "出发/到达时间", "历时", "可否网购", "商务座", "特等座", "一等座", "二等座", "高级软卧", "软卧", "动卧", "硬卧", "软座","硬座", "无座", "其他"]
    table.align["车次"] = "l"
    table.align["出发/到达"] = "l"

    for ticket_list in data.get('data').get('result'):
        field = ticket_list.split('|')
        flag_id = "[身]" if field[18]=='1' else "" # 是否支持身份证
        flag_from = "[始]" if field[4] == field[6] else "[过]" # 是否始发站
        flag_to = "[终]" if field[5] == field[7] else "[过]"# 是否终点站

        price = {}
        if need_price == True:
            ret = query_ticket_price(field[2],field[16],field[17],field[35],train_date)
            price = ret if ret else {}

        table.add_row([
            field[3] + (Fore.YELLOW + flag_id + Fore.RESET ) ,
            '\n'.join([Fore.LIGHTGREEN_EX + flag_from + name_map.get(field[6]) + Fore.RESET,
                        Fore.LIGHTRED_EX + flag_to + name_map.get(field[7]) + Fore.RESET]),
            '\n'.join([Fore.LIGHTGREEN_EX + field[8] + Fore.RESET,
                        Fore.LIGHTRED_EX + field[9] + Fore.RESET]),
            field[10],
            "是" if field[10] else "否",
            # 商务座
            "{0:s}{1:s}".format(field[32] if field[32] else "--", ("\n" + Style.BRIGHT + price.get('A9')) + Style.RESET_ALL if price.get('A9') else ""),
            # 特等座
            "{0:s}{1:s}".format(field[25] if field[25] else "--", ("\n" + Style.BRIGHT + price.get('P')) + Style.RESET_ALL if price.get('P') else ""),
            # 一等座
            "{0:s}{1:s}".format(field[31] if field[31] else "--", ("\n" + Style.BRIGHT + price.get('M')) + Style.RESET_ALL if price.get('M') else ""),
            # 二等座
            "{0:s}{1:s}".format(field[30] if field[30] else "--", ("\n" + Style.BRIGHT + price.get('O')) + Style.RESET_ALL if price.get('O') else ""),
            # 高级软卧
            "{0:s}{1:s}".format(field[21] if field[21] else "--", ("\n" + Style.BRIGHT + price.get('A6')) + Style.RESET_ALL if price.get('A6') else ""),
            # 软卧
            "{0:s}{1:s}".format(field[23] if field[23] else "--", ('\n' + Style.BRIGHT + price.get('A4')) + Style.RESET_ALL if price.get('A4') else ""),
            # 动卧
            field[33] if field[33] else "--",
            # 硬卧
            "{0:s}{1:s}".format(field[28] if field[28] else "--", ("\n" + Style.BRIGHT + price.get('A3')) + Style.RESET_ALL if price.get('A3') else ""),
            # 软座
            "{0:s}{1:s}".format(field[24] if field[24] else "--", ("\n" + Style.BRIGHT + price.get('A2')) + Style.RESET_ALL if price.get('A2') else ""),
            # 硬座
            "{0:s}{1:s}".format(field[29] if field[29] else "--", ("\n" + Style.BRIGHT + price.get('A1')) + Style.RESET_ALL if price.get('A1') else ""),
            # 无座
            "{0:s}{1:s}".format(field[26] if field[26] else "--", ("\n" + Style.BRIGHT + price.get('WZ')) + Style.RESET_ALL if price.get('WZ') else ""),
            # 其他
            field[22] if field[22] else "--"
            ])

    print (table)

def query_ticket_price(train_no, from_station_no, to_station_no, seat_types, train_date):
    url = "https://kyfw.12306.cn/otn/leftTicket/queryTicketPrice"

    #print (train_no, from_station_no, to_station_no, seat_types, train_date)
    payload = {
        'train_no': train_no,
        'from_station_no': from_station_no,
        'to_station_no': to_station_no,
        'seat_types': seat_types,
        'train_date': train_date,
    }

    try:
        response = requests.get(url, headers=header, params=payload)
        if response.status_code == requests.codes.ok:
            #response.encoding = 'utf-8'
            content = response.text
    except Exception as e:
        print (e)

    data = json.loads(content)

    if data.get('status') == False:
        print ("获取数据失败。")
        return None

    return data.get('data')


if __name__ == "__main__":
    #t12306_init()
    conn = Sqlite3.sqlite3_init()

    date = input("请输入乘车时间(YYYY-MM-DD): ")
    from_station = input("请输入出发车站: ")
    to_station = input("请输入到达车站: ")
    purpose_codes = input("请输入类型(1-成人票): ")
    from_s = to_s = ticket_type = None

    ret = Sqlite3.sqlite3_execute(conn, "SELECT teleCode FROM t12306 WHERE stationName ='{0:s}'".format(from_station))#[0][0]
    from_s = ret[0][0]

    ret = Sqlite3.sqlite3_execute(conn, "SELECT teleCode FROM t12306 WHERE stationName ='{0:s}'".format(to_station))#[0][0]
    to_s = ret[0][0]

    Sqlite3.sqlite3_close(conn)

    if purpose_codes == 1:
        ticket_type = 'ADULT'

    check_left_ticket(date, from_s, to_s, ticket_type, True)
```