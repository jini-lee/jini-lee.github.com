---
layout: post
title: kubernetes deployment
categories: [kubernetes]
tags: [kubernetes, k8s, deployment, replicaset, rollingupdate]
comments: true
---

# Deployment
ReplicaSet의 목적은 파드의 집합을 실행 및 업데이트하고 안정적으로 유지하기 위함이다.
Deployment는 ReplicaSet의 기능과 더불어 파드를 정의하여 선언적으로 어플리케이션을 관리하고 상태를 변경할 수 있다.
kubectl로 실행중인 어플리케이션을 변경했던 ReplicaSet이나 ReplicationController와 달리
Deployment manifest로 선언하고 추상화시켜 이 오브젝트 만으로 무중단으로 어플리케이션의 상태를 변경하고 관리할 수 있다.

## Deployment 선언
Deployment는 크게 레이블 셀렉터, 레플리카 수, 파드 템플릿, 업데이트의 수행방법이 선언되어있다.
아래의 예제를 보면 놀랍도록 간단히 선언하고 생성할 수 있음을 알 수 있다.

[Deployment example: simple app](https://github.com/jini-lee/k8s-practice/tree/master/deployment)

## Deployment 업데이트
kubectl rolling-update를 이용해 업데이트를 수행하고 파드가 업데이트가 될때까지 기다렸던 ReplicationController, ReplicaSet과 달리 Deployment는 미리 선언한 정의대로 손쉽게 업데이트 할 수 있다.
Deployment의 업데이트 전략은 기본적으로 RollingUpdate이다. 이는 새로운 파드는 생성하고 기존 파드를 삭제하며 점진적으로 새로운 파드로 트래픽을 유입시키는 방식이다.
업데이트 전략중 Recreate도 있는데 이는 짧은 다운타임이 발생하므로 상황에 맞을 때 사용해야 한다. 이 전략은 가령 새로 수정된 API가 기존 프론트엔드와는 호환되지 않을 때 즉 병렬로 기존 파드와 새로운 파드를 사용할 수 없을 때 선택할 수 있는 전략이다.
업데이트는 빠르게 진행된다. 하지만 minReadySeconds를 사용하면 롤아웃 속도를 늦출 수 있어 해당 과정을 잘 모니터링할 수 있다. 위 `Deployment example: simple app`에서 확인할 수 있다.
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
