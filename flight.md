# 携程机票信息爬取

爬取地址为[携程旅行的国内/国际机票购买](https://flights.ctrip.com/itinerary/oneway/kmg-ckg?date=2020-12-12)网站；爬取内容为从当前到未来一个月内的由北京到重庆、昆明到重庆的机票信息。

## 爬虫使用说明

考虑到机票信息在动态更新，实时性较强，另外爬取任务简单，因此使用了python原生的requests库来编写爬虫脚本。  
该爬虫脚本使用到的库为：`requests`, `json`, `datetime`, `datetime.timedelta`, `fake_useragent.UserAgent`; python版本为`Python 3.8.3` ;  
在安装好相应的库后直接运行*flightInfo.py*即可开始爬取，爬取到的数据保存在同级目录下的*info.json*文件。

## 数据介绍  
从浏览器查看Responce，包含了某趟航班的完整信息:（数据太长，复制了一部分，其余部分为改签、托运、意外险等信息）
```json
{
  "data": {
    "version": "V2",
    "error": null,
    "routeList": [
      {
        "routeType": "Flight",
        "nearRouteCity": null,
        "legs": [
          {
            "flightId": "328077870",
            "legType": "Flight",
            "flight": {
              "id": "328077870",
              "flightNumber": "MF8455",
              "sharedFlightNumber": "",
              "sharedFlightName": null,
              "airlineCode": "MF",
              "airlineName": "厦门航空",
              "craftTypeCode": "738",
              "craftKind": "M",
              "craftTypeName": "波音738",
              "craftTypeKindDisplayName": "中型",
              "specialCraft": false,
              "departureAirportInfo": {
                "cityTlc": "BJS",
                "cityName": "北京",
                "airportTlc": "PKX",
                "airportName": "大兴国际机场",
                "terminal": {
                  "id": 2942,
                  "name": "",
                  "shortName": ""
                }
              },
              "arrivalAirportInfo": {
                "cityTlc": "CKG",
                "cityName": "重庆",
                "airportTlc": "CKG",
                "airportName": "江北国际机场",
                "terminal": {
                  "id": 1508,
                  "name": "T3",
                  "shortName": "T3"
                }
              },
              "departureDate": "2020-12-03 20:10:00",
              "delayedTime": null,
              "arrivalDate": "2020-12-03 23:20:00",
              "comfort": null,
              "punctualityRate": "100%",
              "mealType": "None",
              "mealFlag": true,
              "oilFee": 0,
              "tax": 50,
              "durationDays": 0,
              "stopTimes": 0,
              "stopInfo": null
            },
            "cabins": [
              {
                "id": "2420590390",
                "transitCabinIds": null,
                "combinedCabinIds": null,
                "combinedCabinPriorities": null,
                "pid": "MAgDEAsYwLgCKAA4AECAgICAgICAgAhQAGgBeACIAb7vseMBmAEA",
                "cabinClass": "Y",
                "priceClass": "R",
                "classAreaCode": "Y",
                "classCodeCombine": null,
                "classCodeDesc": null,
                "saleType": "PriorityPackage",
                "ifFreeDelivery": false,
                "directFlightChannel": "",
                "price": {
                  "compositionPrice": false,
                  "price": 400,
                  "salePrice": 400,
                  "printPrice": 400,
                  "fdPrice": 790,
                  "rate": 0.21,
                  "discount": null,
                  "serviceCharge": null,
                  "originalPrice": null,
                  "discountAmount": null,
                  "discountShowType": 0,
                  "discountLabel": null,
                  "favorablePrice": 0,
                  "pcPrice": null
                },
                "seatCount": 10,
                "provideBillType": "Itinerary",
                "groupType": "Priority",
              
```
可以提取主要的信息作为爬虫结果：
- **出发地信息**（出发城市，机场，航站楼，起飞时间）
- **到达地信息**（到达城市，机场，航站楼，落地时间）
- **飞机信息**（航空公司名字，飞机型号，航班号）
- **机票价格**  

相应的爬虫代码如下：


```python
import requests
import json
import datetime
from datetime import timedelta
from fake_useragent import UserAgent

#日期生成器，以start_date为初始值、步长为1逐个返回日期
def gen_dates(start_date, day_counts):
    next_day = timedelta(days=1) 
    for i in range(day_counts):  
        yield start_date + next_day * i

#生成日期列表，从start_date到days_count天后的时间列表
def get_date_list(start_date,days_count):
    #若设置的start_date在当前日期之前，则将start_date设置为当前时间
    if start_date < datetime.datetime.now():
        start = datetime.datetime.now()
    else:
        start = start_date
    # 爬取未来days_count天的机票
    end = start + datetime.timedelta(days=days_count)  
    data = []
    for d in gen_dates(start, ((end - start).days)):
        data.append(d.strftime("%Y-%m-%d"))
    return data

#设置航班的出发和降落地，该列表设置了北京->重庆、昆明->重庆的
cities_data = [
    {"dcity": "BJS", "acity": "CKG", "dcityname": "北京", "acityname": "重庆", "date": "2020-12-12", "dcityid": 1, "token": "6cffa86335703cfaadefc79c4155dfde"},
    {"dcity": "KMG", "acity": "CKG", "dcityname": "昆明", "acityname": "重庆", "date": "2020-12-12", "dcityid": 34, "token": "f80c15c5f37897b802f14871411bcf59"},
    ]
#contents列表用于保存爬取到的、清洗后的数据对象，初始化为空
contents=[]

if __name__ == "__main__":
    #将开始日期设置为2020年12月2日，爬取的日期跨度为30天
    start_date = datetime.datetime.strptime("2020-12-02", "%Y-%m-%d") 
    date_data = get_date_list(start_date,30)
    for city_data in cities_data:
        for day in date_data:
            #在具体查询机票信息的时候需要在URL里设置出发城市、到达城市、日期
            #例如"https://flights.ctrip.com/itinerary/oneway/kmg-ckg?date=2020-12-12"
            url = "https://flights.ctrip.com/itinerary/api/12808/products/oneway/{},{}-xmn?date={}".format(city_data.get('dcity'),
                                                                                                           city_data.get('dcity'),
                                                                                                           day)

            headers = {
                #构造随机请求的Header
                'User-Agent': '{}'.format(UserAgent().random),
                'Referer': 'https://flights.ctrip.com/itinerary/oneway/{},{}-xmn?date={}'.format(city_data.get('dcity'),
                                                                                                 city_data.get('dcity'),
                                                                                                 day),
                "Content-Type": "application/json"
            }
            #构造http request的payload body
            request_payload = {
                "flightWay": "Oneway",
                "classType": "ALL",
                "hasChild": False,
                "hasBaby": False,
                "searchIndex": 1,
                "airportParams": [
                    {"dcity": "{}".format(city_data.get('dcity')),
                     "acity": "CKG",
                     "dcityname": "{}".format(city_data.get('dcityname')),
                     "acityname": "重庆",
                     "date": "{}".format(day),
                     "dcityid": "{}".format(city_data.get('dcityid'))}
                ],
                "token": "{}".format(city_data.get('token'))
            }

            # 发送post请求，将request_payload对象转化为JSON
            response = requests.post(url, data=json.dumps(request_payload), headers=headers, timeout=30).text  
            # 将responce的JSON转为List对象
            routeList = json.loads(response).get('data').get('routeList')
            # 遍历List对象
            for route in routeList:
                # 在内容不为空的时候进行处理
                if len(route.get('legs')) == 1:
                    legs = route.get('legs')
                    flight = legs[0].get('flight')
                    # 提取指定的的信息
                    #飞机信息
                    airlineName = flight.get('airlineName')
                    flightNumber = flight.get('flightNumber')
                    craftTypeName = flight.get('craftTypeName')
                    #出发地信息
                    departureCityName = flight.get('departureAirportInfo').get('cityName')
                    departureAirportName = flight.get('departureAirportInfo').get('airportName')
                    departureterminal = flight.get('departureAirportInfo').get('terminal').get('name')
                    departureDate = flight.get('departureDate')
                    #目的地信息
                    arrivalCityName = flight.get('arrivalAirportInfo').get('cityName')
                    arrivalAirportName = flight.get('arrivalAirportInfo').get('airportName')
                    arrivalterminal = flight.get('arrivalAirportInfo').get('terminal').get('name')
                    arrivalDate = flight.get('arrivalDate')
                    #机票价格（经济舱）
                    cabins = legs[0].get('cabins')[0]
                    price = cabins.get('price').get('price')

                    
                    #将上面获取到的信息存入contents对象
                    contents.append({'airlineName':airlineName,'flightNumber':flightNumber,
                                     'craftTypeName':craftTypeName,'departureCityName':departureCityName,
                                     'departureAirportName':departureAirportName,'departureterminal':departureterminal,
                                     'departureDate':departureDate,'arrivalCityName':arrivalCityName,'arrivalAirportName':arrivalAirportName,
                                     'arrivalterminal':arrivalterminal,'arrivalDate':arrivalDate,'price':price})
                else:
                    pass
    #创建info.json文件保存爬取结果；将contents列表转换为JSON
    with open("info.json", "w+") as f:
        json.dump(contents, fp=f, ensure_ascii = False, indent = 4)
        f.close()
```

## 爬取结果

JSON(部分)：
```json
[
    {
        "airlineName": "厦门航空",
        "flightNumber": "MF8455",
        "craftTypeName": "波音738",
        "departureCityName": "北京",
        "departureAirportName": "大兴国际机场",
        "departureterminal": "",
        "departureDate": "2020-12-03 20:10:00",
        "arrivalCityName": "重庆",
        "arrivalAirportName": "江北国际机场",
        "arrivalterminal": "T3",
        "arrivalDate": "2020-12-03 23:20:00",
        "price": 440
    },
    {
        "airlineName": "厦门航空",
        "flightNumber": "MF5188",
        "craftTypeName": "空客320",
        "departureCityName": "北京",
        "departureAirportName": "首都国际机场",
        "departureterminal": "T3",
        "departureDate": "2020-12-03 10:25:00",
        "arrivalCityName": "重庆",
        "arrivalAirportName": "江北国际机场",
        "arrivalterminal": "T2",
        "arrivalDate": "2020-12-03 13:45:00",
        "price": 1610
    },
    {
        "airlineName": "厦门航空",
        "flightNumber": "MF5203",
        "craftTypeName": "空客321",
        "departureCityName": "北京",
        "departureAirportName": "首都国际机场",
        "departureterminal": "T3",
        "departureDate": "2020-12-03 16:50:00",
        "arrivalCityName": "重庆",
        "arrivalAirportName": "江北国际机场",
        "arrivalterminal": "T2",
        "arrivalDate": "2020-12-03 20:15:00",
        "price": 1060
    },
    {
        "airlineName": "厦门航空",
        "flightNumber": "MF8781",
        "craftTypeName": "波音738",
        "departureCityName": "北京",
        "departureAirportName": "大兴国际机场",
        "departureterminal": "",
        "departureDate": "2020-12-03 16:50:00",
        "arrivalCityName": "重庆",
        "arrivalAirportName": "江北国际机场",
        "arrivalterminal": "T3",
        "arrivalDate": "2020-12-03 20:15:00",
        "price": 500
    },
    {
        "airlineName": "厦门航空",
        "flightNumber": "MF5074",
        "craftTypeName": "空客320",
        "departureCityName": "北京",
        "departureAirportName": "首都国际机场",
        "departureterminal": "T3",
        "departureDate": "2020-12-03 22:10:00",
        "arrivalCityName": "重庆",
        "arrivalAirportName": "江北国际机场",
        "arrivalterminal": "T2",
        "arrivalDate": "2020-12-04 01:10:00",
        "price": 1060
    },
    {
        "airlineName": "厦门航空",
        "flightNumber": "MF5200",
        "craftTypeName": "空客319",
        "departureCityName": "北京",
        "departureAirportName": "首都国际机场",
        "departureterminal": "T3",
        "departureDate": "2020-12-03 06:40:00",
        "arrivalCityName": "重庆",
        "arrivalAirportName": "江北国际机场",
        "arrivalterminal": "T2",
        "arrivalDate": "2020-12-03 11:40:00",
        "price": 1170
    },
    {
        "airlineName": "山东航空",
        "flightNumber": "SC1156",
        "craftTypeName": "波音738",
        "departureCityName": "北京",
        "departureAirportName": "首都国际机场",
        "departureterminal": "T3",
        "departureDate": "2020-12-03 21:25:00",
        "arrivalCityName": "重庆",
        "arrivalAirportName": "江北国际机场",
        "arrivalterminal": "T3",
        "arrivalDate": "2020-12-04 00:40:00",
        "price": 400
    },
    {
        "airlineName": "深圳航空",
        "flightNumber": "ZH3827",
        "craftTypeName": "空客320",
        "departureCityName": "北京",
        "departureAirportName": "首都国际机场",
        "departureterminal": "T3",
        "departureDate": "2020-12-03 10:25:00",
        "arrivalCityName": "重庆",
        "arrivalAirportName": "江北国际机场",
        "arrivalterminal": "T2",
        "arrivalDate": "2020-12-03 13:45:00",
        "price": 1610
    },
    {
        "airlineName": "东方航空",
        "flightNumber": "MU2865",
        "craftTypeName": "空客321",
        "departureCityName": "北京",
        "departureAirportName": "大兴国际机场",
        "departureterminal": "",
        "departureDate": "2020-12-03 08:00:00",
        "arrivalCityName": "重庆",
        "arrivalAirportName": "江北国际机场",
        "arrivalterminal": "T3",
        "arrivalDate": "2020-12-03 10:55:00",
        "price": 500
    },
    {
        "airlineName": "东方航空",
        "flightNumber": "MU6681",
        "craftTypeName": "空客321",
        "departureCityName": "北京",
        "departureAirportName": "大兴国际机场",
        "departureterminal": "",
        "departureDate": "2020-12-03 07:00:00",
        "arrivalCityName": "重庆",
        "arrivalAirportName": "江北国际机场",
        "arrivalterminal": "T3",
        "arrivalDate": "2020-12-03 09:55:00",
        "price": 500
    },
    {

```
