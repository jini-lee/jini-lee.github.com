---
layout: post
title: kubernetes deployment
categories: [kubernetes]
tags: [kubernetes, k8s, deployment, replicaset, rollingupdate]
comments: true
---

# Deployment
ReplicaSet의 목적은 파드의 집합을 실행 및 업데이트하고 안정적으로 유지하기 위함이다. <br>
Deployment는 ReplicaSet의 기능과 더불어 파드를 정의하여 선언적으로 어플리케이션을 관리하고 상태를 변경할 수 있다. <br>
kubectl로 실행중인 어플리케이션을 변경했던 ReplicaSet이나 ReplicationController와 달리 <br>
Deployment manifest로 선언하고 추상화시켜 이 오브젝트 만으로 무중단으로 어플리케이션의 상태를 변경하고 관리할 수 있다.

## Deployment 선언
Deployment는 크게 레이블 셀렉터, 레플리카 수, 파드 템플릿, 업데이트의 수행방법이 선언되어있다.  
아래의 예제를 보면 놀랍도록 간단히 선언하고 생성할 수 있음을 알 수 있다.  

[Deployment example: simple app](https://github.com/jini-lee/k8s-practice/tree/master/deployment)

## Deployment 업데이트
kubectl rolling-update를 이용해 업데이트를 수행하고 파드가 업데이트가 될때까지 기다렸던  
ReplicationController, ReplicaSet과 달리 Deployment는 미리 선언한 정의대로 손쉽게 업데이트 할 수 있다.  
Deployment의 업데이트 전략은 기본적으로 RollingUpdate이다. 이는 새로운 파드는 생성하고 기존 파드를 삭제하며 점진적으로 새로운 파드로 트래픽을 유입시키는 방식이다.  
업데이트 전략중 Recreate도 있는데 이는 짧은 다운타임이 발생하므로 상황에 맞을 때 사용해야 한다.  
이 전략은 가령 새로 수정된 API가 기존 프론트엔드와는 호환되지 않을 때 즉 병렬로 기존 파드와 새로운 파드를 사용할 수 없을 때 선택할 수 있는 전략이다.  
업데이트는 빠르게 진행된다. 하지만 minReadySeconds를 사용하면 롤아웃 속도를 늦출 수 있어 해당 과정을 잘 모니터링할 수 있다. 이는 위 `Deployment example: simple app`에서 확인할 수 있다.  
업데이트를 실행하게되면 기본 전략인 RollingUpdate방식대로 기존 파드는 스케일 다운하고 새로운 파드는 스케일 업하며 점차적으로 새로운 파드를 교체하는 것을 확인 할 수있다.

```
[kubectl describe deployment simpleapp 명령어를 통해 확인한 업데이트 이벤트]

Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  16m    deployment-controller  Scaled up replica set simpleapp-79b55dfbc4 to 3
  Normal  ScalingReplicaSet  7m18s  deployment-controller  Scaled up replica set simpleapp-699dcbcb87 to 1
  Normal  ScalingReplicaSet  7m3s   deployment-controller  Scaled down replica set simpleapp-79b55dfbc4 to 2
  Normal  ScalingReplicaSet  7m3s   deployment-controller  Scaled up replica set simpleapp-699dcbcb87 to 2
  Normal  ScalingReplicaSet  6m48s  deployment-controller  Scaled down replica set simpleapp-79b55dfbc4 to 1
  Normal  ScalingReplicaSet  6m48s  deployment-controller  Scaled up replica set simpleapp-699dcbcb87 to 3
  Normal  ScalingReplicaSet  6m37s  deployment-controller  Scaled down replica set simpleapp-79b55dfbc4 to 0
```

## Deployment 롤백
`kubectl rollout history deployment simpleapp` 명령어를 통해 확인해보면 revision history 확인할 수 있다.  
업데이트된 버전에 문제가 생겼을 경우 revision 버전을 명시해 롤백을 하거나 이전 버전으로 되돌릴 수 있다. 이 점이 Deployment의 강력한 장점중 하나이다.  

```
$ kubectl rollout undo deployment simpleapp # 이전 버전으로 롤백
$ kubectl rollout undo deployment simpleapp --to-revision=1 # 특정 버전으로 롤백
```

## Deployment 전략
[문제가 있는 어플리케이션](https://hub.docker.com/r/zaddyjini/unhealthy)를 배포하게된 경우에 Deployment 전략으로 문제를 회피할 수 있을지 설명하려 한다.  
minReadySeconds, maxSurge, maxUnavailable, readinessProbe 개념과 함께 설명하겠다.  
우선 readinessProbe는 [해당글](https://jini-lee.github.io/kubernetes/2020/08/19/probe/)을 참고하면된다.  
앞서 롤아웃을 느리게 만드는 minReadySeconds는 사실 업데이트 실행 시 새로운 파드가 교체될 때 해당 파드가 정상적으로 동작하는지 확인하기 위해 최소 보장 시간이다. 즉 minReadySeconds가 20으로 설정되어 있으면 새로운 파드가 생성되고 20초간 기다린 후 롤아웃이 재개되는 것이다. 하지만 기다리기만 하면 되는 것이 아니다. 파드가 정상적으로 실행되는지 확인을 해야한다. 이를위해 probe를 사용하며, 이용하지 않는다면 문제있는 파드를 계속 업데이트 해나갈 것이다. 이러한 상황을 방지하기 위해 파드 health check를 하고 정상 실행이 가능한 상태라면 앤드포인트가 노출되는 것이다.  
정상적으로 잘 업데이트가 된다면 문제가 없겠지만 문제가 있는 어플리케이션 배포 시 업데이트를 중단시켜야 하는데 이때 사용할 수 있는 설정값이 maxUnavailable과 maxSurge이다. maxUnavilable은 의도하는 replica 수에 사용하지 못하는 파드의 최대 수 또는 비율(비율계산 후 내림)이고, maxSurge는 의도하는 replica 수에 허용되는 최대 파드 수 또는 비율(비율계산 후 반올림)이다. 예를들어 replicas가 3이고 maxSurge가 1이면 롤아웃 시 허용되는 최대 파드 수는 4이며 maxUnavailable이 1이면 파드의 수는 항상 2이상으로 유지해야한다.(단, maxSurge 1로 인해 4개 이하)
비율과 절대값을 쓸 수 있는데 절대값으로 예를들어 설명하겠다. 아래의 예제는  Deployment에 replica를 3으로 설정하고 maxSurge는 1, maxUnavailable은 0, minReadySeconds가 20인 Deployment가 있다. 새로운 이미지를 교체하는 Deployment를 업데이트(v1->v2)한다면 v1의 replica는 3으로 되어있을 것이고 maxSurge 1에 의해 새로운 이미지를 가진 파드가 하나 생성된다. 이때 minReadySeconds에 의해 20초간 기다릴것이고 ReadinessProbe에 의해 health check를 한다. 이때 probe에 의해 파드가 문제가 있음을 발견하고 ready상태로 남아 앤드포인트를 노출하지 않는다. 이때 동작하지 않는 파드가 이미 하나 있고 replica 수가 4이며 maxUnavailable 0이므로 롤아웃이 더 이상 진행되지 않을 것이다.(의도한 replica 수 3이라 3보다 더 작아지면 안되므로 기존 파드를 삭제하지 않음)  
따라서 문제가 있는 어플리케이션 배포를 위 전략으로 막을 수 있는 것이다. 기본적으로 롤아웃은 10분동안 진행되지 않으면 실패한 것으로 간주하는데 progressDeadlineSeconds로 시간을 설정할 수 도 있다.


[Deployment strategy example](https://github.com/jini-lee/k8s-practice/tree/master/deployment/strategy)
