---
layout: post
title: "monitoring - grafana deployments"
subtitle: "모니터링: 그라파나 실행과 설정"
category: monitoring
---

### 4. deploy Grafana

#### docker-compose.monitor.yml

그라파나는 프로메테우스가 수집한 시계열 데이터(metric)를 시각화해주는 역할을 수행한다. 이전 글에서 Prometheus를 `/Prometheus/` 경로를 통해 reverse proxy 했듯, Grafana 또한 `/Grafana/` 경로를 통해 reverse proxy할 예정이다.

이를 위해서는 당연히 웹서버인 Nginx의 설정 파일과 Grafana의 설정파일인 `defaults.ini`를 수정해야 한다. 나는 `defaults.ini`를 컨테이너 외부의 `grafana.ini`로 연결해두었다.

> You can manage data sources in Grafana by adding YAML configuration files in the provisioning/datasources directory. Each config file can contain a list of datasources to add or update during startup. If the data source already exists, Grafana reconfigures it to match the provisioned configuration file.

```yml
services:
  prometheus:
    container_name: prometheus
    image: prom/prometheus
    restart: unless-stopped
    volumes:
      - type: bind
        source: /home/monitoring/prometheus/prometheus.yml
        target: /etc/prometheus/prometheus.yml
      - /home/monitoring/prometheus/data:/prometheus/data
    command: # web.enalbe-lifecycle은 api 재시작없이 설정파일들을 reload 할 수 있게 해줌
      - "--web.enable-lifecycle"
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
      - type: bind
        source: /home/monitoring/grafana/grafana.ini
        target: /usr/share/grafana/conf/defaults.ini
      - /home/monitoring/grafana/datasources:/etc/grafana/provisioning/datasources
    ports:
      - 19092:3000
```

#### grafana.ini

reverse proxy를 위해 `grafana.ini`에서 손 봐야할 부분은 크게 세 군데이다. `[server]` 부분의 `http_port`, `domain`, `root_url`을 본인 서비스에 맞추어 수정해주면 된다. 나는 port는 변경하지 않았지만, 별도로 사용하는 도메인이 있어 수정해주었다. 아래의 localhost는 예시이므로, 본인의 도메인(example.com)을 적으면 된다. 이어 root_url도 본인이 reverse proxy 하고자 하는 경로에 맞추어 수정해주면 된다. 기본 값은 `%(protocol)s://%(domain)s:%(http_port)s/` 일 것이다.

```ini
# The ip address to bind to, empty will bind to all interfaces
http_addr =

# The http port to use
http_port = 3000

# The public facing domain name used to access grafana from a browser
domain = localhost

# Redirect to correct domain if host header does not match domain
# Prevents DNS rebinding attacks
enforce_domain = false

# The full public facing url
root_url = %(protocol)s://%(domain)s:%(http_port)s/grafana/
```

#### nginx.conf

사용 중인 nginx의 일부이다. `Grafana`는 위에서 설명했듯 `/grafana/` 경로로 리버스 프록시 되어 접근한다.

This is part of the nginx configuration in use. Grafana is reverse proxied to the server through the configured `/grafana/` path.

> `grafana`는 설정 파일에서 upstream 블록으로 정의되어 있다.
>
> `grafana` are defined as upstream blocks in the file.

```
  location /grafana/ {
    proxy_pass 	http://grafana;
	  proxy_set_header Host $host;
	  rewrite  ^/grafana/(.*)  /$1 break;
  }

  # Proxy Grafana Live WebSocket connections.
  location /grafana/api/live/ {
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_set_header Host $host;
    proxy_pass http://grafana;
    rewrite  ^/grafana/(.*)  /$1 break;
  }
```
