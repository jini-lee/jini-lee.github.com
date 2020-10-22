---
layout: post
title: 여러개의 container를 가지는 Pod 
categories: [kubernetes]
tags: [kubernetes, k8s, Pod]
comments: true
---


### Pod란
Pod는 kubernetes에 의해 배포될 수 있는 가장 작은 유닛이다. 이는 하나의 container를 kubernetes에 실행하기 위해 반드시 하나의 Pod가 필요하다는 뜻이다. 동시에 Pod는 여러개의 container를 가질 수 있다. 하나의 Pod안의 container들은 어떤 목적을 위해 강하게 결합된 것들이다. 이러한 특징을 단순히 사용하기보다 깊이 이해하고 사용한다면 kubernetes 운영환경에서 서비스를 설계하는데 큰 도움이 될 것이다.

### Kubernetes를 만든이들은 Pod가 왜 여러개의 container를 가질 수 있게 하였을까  

Kubernetest는 container를 관리하기위해 restart policy, live probe와 같은 정보를 가지고 있다. 이러한 정보들을 각 container마다 설정하기보다 통합된 entity로 관리한다면 이득을 얻게 될 것이다.
Pod의 container들은 같은 logical host이다. 같은 network namespace를 사용하고 볼륨을 공유 할 수 있다. 이런 특징을 가진 container들은 효율적인 통신을 할 것이고 상호간의 data locality를 보장한다. 
그렇다면 Pod와 마찬가지로 하나의 container가 여러 기능을 담으면 될거 같지만 그렇지 않다. 이유는 많은 기능들을 하나의 container에 담는 것은 하나의 container는 하나의 Process를 가지는 전략에 반한다. 이것은 __SOLID__의 __단일책임의 원칙(SRP)__과 결이 비슷하다고 생각한다. 왜냐면 서비스를 뒷받침하는 여러 소프트웨어들의 디펜던시의 결합성을 낮출 수 있고(decoupling), 또한 협업 시 세분화된 container는 재상용성이 좋다.
물론 하나의 Pod가 여러 container를 가지는 것, 하나의 container는 하나의 프로세스를 가지는 것에도 trade-off는 존재할 것이다. 적어도 이런 특징을 깊이 고민한 후 설계한다면 더 나은 소프트웨어가 만들어지지 않을까.

### 여러 container를 가지는 Pod의 예

Pod가 여러 container를 가지는 주요 목적은 주요 기능을 위해 특정 container가 다른 container를 헬퍼 프로세스로써 동작하기 위함이다. 
- Sidecar container는 main container를 도와준다. 예컨데 main conatiner(웹 서버)가 뱉어내는 로그를 모니터링(이 방식은 잘 쓰이진 않는다. cluster-lever logging)하는 conatiner, 생성된 파일이나 데이터를 주요 container에 로드하는 container가 있다(shared volume). 이런 일종의 helper container를 빌드해둔다면 이런 역할이 필요한 conatiner에 사용할 수 있으므로 재사용성이 높다.   
- Proxy container는 main container에 연결된다. 예컨데 nginx container는 static file들을 client에 서빙하고 main container(웹 서버)의 reverse proxy 역할을 하기도 한다. 

### Pod의 container간 커뮤니케이션 방식

- Shared volumes 
Pod의 container들은 같은 호스트에 존재하므로 볼륨을 공유할 수 있다. 하지만 Pod는 stateless라 Pod가 재시작되거나 종료되면 이 볼륨의 상태 또한 사라진다. 이 점을 기억해야 할 것이다. 데이터를 영구적으로 보존하기 위해선 [Persistent Volume](https://jini-lee.github.io/kubernetes/2020/07/16/volume-2/)을 사용해야한다.
- Inter-process communications (IPC)
Pod의 container들은 같은 IPC namespace를 가진다. container들은 표준 inter-process 통신을 할 수 있다. 가령 System V semaphore나 POSIX shared memory. 표준 Linux message queue를 이용해 producer container와 consumer container를 가지는 Pod를 설계할 수 도 있다.

```
apiVersion: v1
kind: Pod
metadata:
  name: std_linux_message_queue
spec:
  containers:
  - name: 1st
    image: allingeek/ch6_ipc
    command: ["./ipc", "-producer"]
  - name: 2nd
    image: allingeek/ch6_ipc
    command: ["./ipc", "-consumer"]
```

### 덧붙임

- 하나의 Pod속 container들은 어떻게 expose될까? 위에서 말한대로 같은 Pod의 container들은 같은 IP, port space를 가지는데, 각각이 다른 포트를 가짐으로써 expose될 수 있다.
- 하나의 Pod의 container들이 실행될 때 이들은 병렬적으로 실행된다. 일반적으로 실행 우선순위를 줄 수 없다. 위 linux message 예제의 Pod는 두번째 container가 먼저 실행된다면 메시지가 큐에 없는 상태로 생성되므로 정상 실행이 되지 않을것이다. 이를 회피하기 위해선 [Init Container](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)를 참고해야 manifest를 작성해야한다.   
