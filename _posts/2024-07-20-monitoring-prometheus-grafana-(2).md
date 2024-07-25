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

이제 Prometheus와 Grafana를 실행해보자.
