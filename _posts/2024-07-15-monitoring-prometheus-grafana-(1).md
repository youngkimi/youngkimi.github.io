---
layout: post
title: "monitoring - prometheus, grafana (1)"
category: monitoring
---

### Note

개발 완료한 백엔드 서버의 성능을 측정, 개선하기 위해서 모니터링 프로그램인 [프로메테우스](https://prometheus.io/docs/introduction/overview/)와 [그라파나](https://grafana.com/docs/grafana/latest/)를 사용하였다. 병목을 확인하기 위해서 성능 측정의 대상은 웹 서버 (Nginx), WAS (Spring), DB (MySQL)로 설정하였다.

I used [Prometheus](https://prometheus.io/docs/introduction/overview/) and [Grafana](https://grafana.com/docs/grafana/latest/), monitoring tools, to measure and improve the performance of the backend server. To identify bottlenecks, I set up performance monitoring for the web server (Nginx), WAS (Spring), and DB (MySQL).

### 1. how it works

프로메테우스는 시계열 기반 오픈소스 모니터링 시스템이다. 메트릭을 타임 스탬프와 함께 수집한다. 이런 메트릭을 분석하면 서비스의 현황을 파악하고 문제를 대비할 수 있으며, 문제 발생 시 원인을 파악하고 해결할 수 있다.

그라파나는 프로메테우스를 통해 수집한 시계열 데이터를 시각화해주는 역할을 한다. 물론 프로메테우스만을 위한 프로그램은 아니므로, 그 외의 프로그램(엘라스틱 서치, Influx DB ...)도 데이터 소스로 설정할 수 있다. 프로메테우스를 데이터 소스로 활용 시, `PromQL`을 활용해서 더 복잡한 쿼리와 처리를 수행할 수 있다. 이 외에도 임계 상황에서 알림을 발송하는 역할도 가능하다.'

프로메테우스는 push 방식이 아닌 pull 방식으로 작동한다. 대상이 되는 서비스는 exporter 등을 활용하여 프로메테우스가 접근하여 데이터를 수집할 수 있는 엔드포인트를 마련하고, 프로메테우스는 일정 주기마다 해당 엔드포인트에 접근해 메트릭을 가져온다.

스프링에서는 `Spring-Boot-Actuator`를 활용하여 프로메테우스를 위한 엔드포인트를 마련할 수 있다. 의존성을 주입한 뒤 `/actuator/health`로 접근하면, 서버의 생존 여부를 파악할 수 있다. `/health` 말고도, `Spring-Boot-Actuator`에서 기본적으로 제공하는 [엔드포인트](https://docs.spring.io/spring-boot/docs/2.1.13.RELEASE/reference/html/production-ready-endpoints.html)들이 있으니 확인해보면 된다.

자세한 설정 정보는 다음 글에서 이어 작성하겠다.

Prometheus is a open-source monitoring system based on time-series data. It collects metrics along with timestamps. By analyzing these metrics, we can understand the current state of our service, anticipate issues, and identify and resolve problems when they arise.

Grafana visualizes the time-series metrics collected by Prometheus. It is not limited to Prometheus, though; you can configure other programs (ElasticSearch, InfluxDB, etc.) as data sources. You can simplify more complex queries and data processing with `PromQL`. Grafana also has capability to send alerts in critical situations.

Prometheus operates using a pull-based method rather than a push-based one. The target service set up endpoints that Prometheus can access to collect data. Prometheus periodically accesses to these endpoints to gather metrics.

In the Spring Framework, We can set up endpoints for Prometheus using `Spring Boot Actuator`. After adding the necessary dependencies, accessing `/actuator/prometheus` allows Prometheus to scrape metrics from the server. Besides `/prometheus`, `Spring Boot Actuator` offers other endpoints for various monitoring purposes.

I'll provide the configuration details in the following article.
