---
layout: post
title: Docker container logs 
categories: [docker, container]
tags: [log, container]
comments: true
---



## Docker container의 로깅 시스템
  컨테이너 운영환경에서 Application을 운영하다보면 기본적인 로깅 시스템의 이해가 필수적이다. 도커는 Log streams(도커의 기본 로그는 stderr, stdout만을 다룸)을 컨테이너 엔진에 의해 처리하고 로깅 드라이버로 리디렉션 한다. 이 로그들이 어느 위치에 어떠한 형태로 저장되는지 등의 도커 컨테이너의 로깅 시스템에 대한 매우 기본적인 정리를 하고자 한다. 



### Logging drivers
  Logging driver는 컨테이너들의 log streams를 모으는 역할을 한다. 기본 driver는 json-file이며 syslog, journald(systemd journal용 드라이버), fluentd, logagent 등이 있다. [logging driver는 도커를 실행할 때 daemon.json 파일에 설정할 수 있다.](https://docs.docker.com/config/containers/logging/json-file/)  
  참고로 fluentd를 로깅 드라이버로 쓰려면 컨테이너와 동일한 host에 fluentd daemon이 있어야한다. daemon.json을 아래와 같이 작성하거나 command로도 로그 스트림은 fluentd로 리디렉션된다.  
  [fluentd docker logging 레퍼런스](https://docs.fluentd.org/container-deployment/docker-logging-driver)

```
# daemon.json
 {
   "log-driver": "fluentd",
   "log-opts": {
     "fluentd-address": "fluentdhost:24224"
   }
 }
```

```
# docker command
$ docker run --log-driver=fluentd --log-opt fluentd-address=fluentdhost:24224
```




### Logs location
  JSON log 파일은 linux 환경에서는 /var/lib/docker/containers/ 하위에 존재하고 mac os에서는 ~/Library/Containers/com.docker.docker/Data/ 하위에 존재한다. 이 때 하나의 컨테이너당 하나의 로그 파일을 가지며 container의 id로 파일을 찾으면된다. 아래의 명령을 통해 확인할 수 있는 로그가 언급한 location에서 가져오는 것이다.  

```
$ docker logs <container_id>
```



### 한계점
  기본 설정값만으로도 도커는 로그를 적절히 처리해주는거 같지만 production으로 운영하기에는 여러문제를 가진다.  
  container는 stateless라 컨테이너가 종료되면 로그도 사라진다.  
  log stream을 계속 기록하다보면 disk space가 가득찰 것이다. 이는 다수의 컨테이너를 가지는 클러스의 경우 더 빨리 도래할 것이다. 이 문제는 log rotate(기본 제공기능이 아님)를 이용해 해결할 수 있다. 하지만 오랜기간 보관해야할 로그의 경우는 적절하지 않다.  
  production은 클러스터 레벨에서의 수 많은 컨테이너가 내뿜는 로그를 지속적으로 모니터링하는것이 필수적이다. 따라서 여러 컨테이너의 로그를 저장에 특화된 저장용량이 충분히 큰 스토리지에 모아 모니터링해야한다.  
  위 문제를 해결하기위해 EFK, ELK 등의 스택을 쓰기도하며 kafka를 이용해 log stream을 적절히 streaming하기도 한다.

