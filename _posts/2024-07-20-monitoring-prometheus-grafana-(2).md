---
layout: post
title: "monitoring - prometheus, grafana (2)"
category: monitoring
---

### 2. setting up the exporters

#### spring-actuator.yml

앞서 말한 exporter를 설정한다. 프로메테우스가 사용할 수 있는 endpoint(`/prometheus`)를 노출한다.

- exporter의 엔드포인트로 접속할 수 있는 base-path 설정을 했다. `/system` 경로로 접근하면 필요한 메트릭을 수집할 수 있다. 기본은 `/actuator`이다.
- 메트릭 노출을 위해서는 `enable`, `exposure` 설정 모두를 해주어야 한다.
- 메트릭 중에서는 상당히 민감한 정보(env, info ...)를 포함하는 것도 있으니, 외부 노출에 주의하자.
- Spring Security 등과 함께 사용하여 메트릭 수집 주체의 인가 여부를 항상 확인할 것.
- 이를 위해 admin 권한의 JWT token을 헤더로 같이 전송하였다.

Set up the exporter. Expose the endpoint (`/prometheus`) that Prometheus can use.

- I configured the base path to access the exporter's endpoint. By accessing the `/system` path, the necessary metrics can be collected. The default base path is `/actuator`.
- To expose the metrics, both enable and exposure settings need to be configured.
- Some metrics may include highly sensitive information (e.g., env, info...), so be cautious about exposing them externally.
- Always verify the authorization of the agent collecting the metrics, for instance by using Spring Security.
- To achieve this, I sent an admin JWT token in the header.

> 기본적으로 'shut-down' 엔드포인트를 제외한 모든 엔드포인트가 활성화되어 있다. 그러나 보안을 위해 모든 엔드포인트의 기본값을 비활성화로 설정하고, 필요한 엔드포인트만 선택적으로 활성화하는 화이트리스트 방식을 사용하는 것이 좋다. 주의할 점은 엔드포인트를 활성화하는 것만으로는 사용할 수 없으며, 추가로 노출(expose)시켜야 한다. 웹의 경우에는 '/health' 엔드포인트만 기본적으로 노출되어 있습니다.
>
> By default, all endpoints are enabled except for the 'shut-down' endpoint. However, for security reasons, it is recommended to set all endpoints to disabled by default and use a whitelist approach, selectively enabling only the necessary endpoints. It's important to note that enabling an endpoint alone does not make it usable; it must also be exposed. In the case of `Web`, only the '/health' endpoint is exposed by default.

```yml
management:
  endpoints:
    enabled-by-default: false
    web:
      exposure:
        # set exposure endpoints
        # be cautious about enabling/exposing env endpoints
        # you can use a wildcard, but recommended to avoid them.
        include: health,info,env,metrics,mappings,prometheus
      base-path: "/system"
    # JMX/Web has different default method settings for exposing.
    # If you're not going to use JMX, disable all of  them.
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
    # never: env values are masked as ***.
    # always: all env values are printed.
    env:
      enabled: true
      show-values: never
    # metrics of application.
    metrics:
      enabled: true
    # informations about Git.
    # need to configure additional git settings.
    info:
      enabled: true
    # shows every Request Mapping
    mappings:
      enabled: true
    # shows metrics in formats that Prometheus can interpret
    # need dependencies of  `micrometer-registry-prometheus`
    prometheus:
      enabled: true

  server:
    port: 9090
```

> Actuator는 서버에 대한 다양한 정보를 다루기 때문에 보안에 주의를 기울여야 한다. 관리 포트와 서비스 포트를 분리하고, Actuator에 접근할 때는 Security를 활용하여 인증과 권한을 철저히 검사하는 것이 좋다. 또한, Actuator의 기본 경로를 사용하지 않고 base-path를 변경하여 운영하는 것이 좋다.
>
> Since Actuator provides various information about the server, it's important to prioritize security. You should separate the management port from the service port and use security mechanisms to enforce authentication and authorization when accessing Actuator endpoints. Additionally, consider changing the base path from the default Actuator path to enhance security.

이제 설정한 endpoint들로 `GET`하면 메트릭을 확인할 수 있다.

You can `GET` metrics by accessing the endpoints the you've set up.

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
