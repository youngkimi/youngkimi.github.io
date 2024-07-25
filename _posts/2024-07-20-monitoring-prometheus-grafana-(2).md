---
layout: post
title: "monitoring - prometheus, grafana (2)"
category: monitoring
---

### 2. settings

#### spring-actuator.yml

앞서 말한 exporter의 역할을 한다. 프로메테우스가 사용할 수 있는 endpoint를 설정한다.

- exporter의 엔드포인트로 접속할 수 있는 base-path 설정을 했다. `/system` 경로로 접근하면 필요한 메트릭을 수집할 수 있다.
- 메트릭 노출을 위해서는 `enable`, `exposure` 설정 모두를 해주어야 한다.
- 메트릭 중에서는 상당히 민감한 정보(env, info ...)를 포함하는 것도 있으니, 외부 노출에 주의하자.
- Spring Security 등과 함께 사용하여 메트릭 수집 주체의 인가 여부를 항상 확인할 것.
- 이를 위해 `Admin`의 JWT token을 헤더로 같이 전송하였다.

```yml
management:
  endpoints:
    # 기본적으로는 shut-down 제외 모두 enabled
    # 보안 상의 이유로 default를 false로 설정 후,
    # 사용을 희망하는 endpoint에 대해서만 enable 하자. (화이트리스트)
    # enable 한다고 바로 사용할 수 있는 것은 아니다.
    # expose 해주어야 한다.
    # Web 의 경우는 health 만 기본적으로 expose 되어있다.
    enabled-by-default: false
    web:
      exposure:
        # set exposure endpoints
        # 일반적으로 env endpoint enable/expose 시 주의해야 한다.
        # 학습 프로젝트인만큼 exposure 하겠다.
        # wildcard를 사용할 수 있으나, 사용을 지양한다.
        include: health,info,env,metrics,mappings,prometheus
      base-path: "/system"
    # 앞서 말했듯, JMX/Web은 expose default method의 설정이 다르다.
    # 따라서 JMX를 사용하지 않을 것이라면 JMX는 모조리 disable 하자.
    jmx:
      exposure:
        exclude: "*"
  endpoint:
    # never: status (up, down)
    # always: disk space, path, ping ...
    health:
      # set endpoint enabled
      enabled: true
      show-details: always
    # never: env value는 ***으로 마스킹 되어 출력된다.
    # always: env value 모두 출력된다.
    env:
      enabled: true
      show-values: never
    # 어플리케이션의 메트릭 정보 표시
    metrics:
      enabled: true
    # Git 에 대한 정보를 다룬다.
    # 추가적인 Git 설정이 필요하다.
    info:
      enabled: true
    # RequestMapping의 모든 경로 표시
    mappings:
      enabled: true
    # Prometheus 서버에서 스크랩할 수 있는 형식으로 변환.
    # `micrometer-registry-prometheus`의 dependency가 필요.
    prometheus:
      enabled: true

  server:
    port: 9090
# Actuator는 서버에 대한 다양한 정보를 다룬다. 보안에 주의를 더 기울여야 한다.
# 1. management port와 service port를 분리하자.
# 2. Actuator 접근 시, Security 등을 활용하여 인증과 권한 여부를 검사하자.
# 3. Actuator Default 경로를 사용하지 않고, base-path를 변경하여 운영한다.
```

#### docker-compose.monitor.yml

프로메테우스의 그라파나 컨테이너를 생성할 수 있는 파일. 도커에 대해서 설명하지는 않겠다.

볼륨(bind) 설정을 통해서 내부 설정 파일을 컨테이너 내부로 진입하지 않고서도 수정할 수 있도록 설정하였다. 이후 커맨드 옵션에 `--web.enable-lifecycle`을 설정하여, 컨테이너 재가동 없이 런타임에 바뀐 설정을 바로 적용할 수 있도록 했다.

이후 설정 파일의 위치(`config.file=`), 프로메테우스의 외부 노출 url(`web.route-prefix=`), 내부 route 접두사(`web.route-prefix=`) 등을 설정해주었다. 내부 route 접두사는 nginx 등의 웹 서버를 사용하여 요청 경로를 라우팅하여 경로를 조작하는 경우 (redirect 등을 사용한다거나) 세팅해주어야 한다.

나는 `/prometheus/` 경로를 `/`등으로 strip off 하거나 하지 않았으므로, 외부 경로와 동일하게 설정해주었다. 사실 `web.route-prefix=`와 같은 값으로 기본 설정되므로 나는 설정할 필요가 없었다.

그라파나는 [데이터 소스를 관리할 수 있는](https://grafana.com/docs/grafana/latest/administration/provisioning/) 볼륨만 설정해두었다.

> You can manage data sources in Grafana by adding YAML configuration files in the provisioning/datasources directory. Each config file can contain a list of datasources to add or update during startup. If the data source already exists, Grafana reconfigures it to match the provisioned configuration file.

> 컨테이너가 종료될 때 로그를 유지하기 위해 볼륨 디렉토리를 설정하는 것이 좋다.
>
> It's better to set a volume directory to keep the logs when the container shuts down.

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

#### prometheus.yml

프로메테우스의 설정 정보를 담은 파일. 위에서 경로 설정할 때 적어두었던 그 파일이다.

```yml
global:
  scrape_interval: 15s # 얼마나 자주 스크랩 할 것인가
  scrape_timeout: 10s # 스크랩 타임아웃 시간
  evaluation_interval: 15s # rules (쿼리, 알림) 평가 시간.
alerting: # 알림 매니저 설정은 아직 진행하지 않았다.
  alertmanagers:
    - static_configs:
        - targets: []
      # scheme: http
      # timeout: 10s
      # api_version: v1
scrape_configs:
  - job_name: spring-app # job의 이름.
    honor_timestamps: true # true 설정 시, 타겟에서 제공하는 타임스탬프, false는 prometheus의 타임스탬프를 사용.
    scrape_interval: 15s
    scrape_timeout: 10s
    metrics_path: /system/prometheus
    scheme: http
    static_configs:
      - targets:
          -  # 타겟 도메인
```

메트릭을 수집 주기(scrape_interval), 스크랩의 타임아웃 시간(scrape_timeout)을 전역적으로 설정할 수도, 아래에서 job 별로 설정해줄 수도 있다. scrape_timeout은 scrape_interval 보다 짧아야 한다.

alerting 부분에서 알림 매니저에 관한 설정을 할 수 있다. 모니터링 중 임계치에 도달하면 관련 알림을 전송할 수 있다. email 뿐 아니라, slack, discord 등의 [채널](https://prometheus.io/docs/alerting/latest/configuration/)을 통해서도 알림을 전달할 수 있다.

scrape_configs 에서 스크랩 Job을 정의하고 설정할 수 있다. 메트릭의 대상이 되는 path, 타겟 도메인과 scheme을 지정하고, 전역적으로 설정했던 메트릭 수집 주기, 타임아웃 간격을 Job 마다 설정할 수도 있다.

#### nginx.conf

```
location /system/ {
    proxy_pass      http://release-main-server;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}

location /prometheus/ {
    proxy_pass      http://prometheus;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

사용 중인 nginx의 일부이다.`Prometheus`는 위 설정된 `/system/` 경로로 리버스 프록시 되어 서버에 접근, 메트릭을 수집한다. 아래 `/prometheus/` 경로를 통해 프로메테우스 서버에 접근할 수 있다.

> `release-main-server`, `prometheus`는 설정 파일에서 upstream 블록으로 정의되어 있다.
>
> `release-main-server` and `prometheus` are defined as upstream blocks in the Nginx configuration file.
