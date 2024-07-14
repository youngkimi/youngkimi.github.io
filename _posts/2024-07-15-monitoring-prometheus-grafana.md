---
layout: post
title: "monitoring - prometheus, grafana"
category: monitoring
---

#### Note

개발 완료한 백엔드 서버의 성능을 측정, 개선하기 위해서 모니터링 프로그램인 [프로메테우스](https://prometheus.io/docs/introduction/overview/)와 [그라파나](https://grafana.com/docs/grafana/latest/)를 사용하였다. 병목을 확인하기 위해서 성능 측정의 대상은 웹 서버 (Nginx), WAS (Spring), DB (MySQL)로 설정하였다.

I used Prometheus and Grafana, monitoring tools, to measure and improve the performance of the backend server. To identify bottlenecks, I set up performance monitoring for the web server (Nginx), WAS (Spring), and DB (MySQL).

### 1. how it works

### 2. docker-compose

```yml
services:
  prometheus:
    container_name: prometheus
    image: prom/prometheus
    restart: unless-stopped
    volumes:
      - type: bind
        source: ./prometheus/prometheus.yml
        target: /etc/prometheus/prometheus.yml
    command:
      - "--web.enable-lifecycle" # enable config files reloaded without api restart
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--web.external-url=/prometheus/"
      - "--web.route-prefix=/prometheus/"
    ports:
      - 19091:9090

  grafana:
    container_name: grafana
    image: grafana/grafana
    restart: unless-stopped
    volumes:
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
    ports:
      - 19092:3000
```

### 3. 繁体中文测试

This is a chinese test post to show you how chinese is displayed.

善我王上魚、產生資西員合兒臉趣論。畫衣生這著爸毛親可時，安程幾？合學作。觀經而作建。都非子作這！法如言子你關！手師也。

以也座論頭室業放。要車時地變此親不老高小是統習直麼調未，行年香一？

就竟在，是我童示讓利分和異種百路關母信過明驗有個歷洋中前合著區亮風值新底車有正結，進快保的行戰從：弟除文辦條國備當來際年每小腳識世可的的外的廣下歌洲保輪市果底天影；全氣具些回童但倒影發狀在示，數上學大法很，如要我……月品大供這起服滿老？應學傳者國：山式排只不之然清同關；細車是！停屋常間又，資畫領生，相們制在？公別的人寫教資夠。資再我我！只臉夫藝量不路政吃息緊回力之；兒足灣電空時局我怎初安。意今一子區首者微陸現際安除發連由子由而走學體區園我車當會，經時取頭，嚴了新科同？很夫營動通打，出和導一樂，查旅他。坐是收外子發物北看蘭戰坐車身做可來。道就學務。

國新故。

> 工步他始能詩的，裝進分星海演意學值例道……於財型目古香亮自和這乎？化經溫詩。只賽嚴大一主價世哥受的沒有中年即病行金拉麼河。主小路了種就小為廣不？

From [亂數假文產生器 - Chinese Lorem Ipsum.](http://www.richyli.com/tool/loremipsum/)

### 4. 简体中文测试

效育声去本义然空，各值太法心想，场强实地。 题铁习点儿表管少间千，只何政亲织文意部，千影画派证男须。 手反取长风治增非等直难群，连取及天他己事头级，影数弦适把气快目人。 专议以省通引而千个，格则口段度样水热马，地教少务改磨。 包思外心半院应她算斯，市外会快记路又火学，劳如肃它准众丧边。

> 团算部住县单总边素格军所，合音府教看和广光采率位转，位用品根确针百。 证其标元角工方海接交他，论象切万世认一响义，治然身本风弦带题。 向我次路持加北，她不反心。 说总元军例市决，现始即算证养，规走还壳。

因林可相儿应满军，热影省条律因资再，整肃赤心将届。 局广写两量备验还，南教事争工民的，备进研上布。 素身电活非直，速这区交示从，百层达。 资量那毛什京身，白这快。 半打容三手开常价或，手严量般象式效，名可重芽门适。 来设什一我么，光界美么或，住身式准。 造酸改表委验众办地百养，商物战众本列听度名院，制压录丽快与千机内。 住需当四议决得命南然照按民置，当住命形金决否矿单外。 气象理离开新集增际，三划方工义很年关，拉许准孝口。 构片出干计由备美打养，持育总指承入无己。

From [假文生成器， lorem ipsum Chinese](http://www.cancms.com/content/dummytext)
