本文针对k8s v1.18 or higher

# 一、影响节点调度的几个因素
## 1.Pod资源限制，如CPU,MEM

## 2.选择器标签，如nodeselector

## 3.软、硬亲和性(nodeaffinity)
> 软：尝试满足，不保证
> 硬：约束条件必须满足

## 4.污点 (taint)
> 默认master是污点节点，即不能跑业务pod



# 二、什么是controller

> 在集群上管理和运行容器的对象

## 1. controller 与 Pod关系
> Pod是通过controller实现应用的运维，如伸缩，滚动升级

## 2.Controller 与 Pod 关联
--> 通过 label  ： selector

## 3. Deployment应用场景与功能

> web服务，微服务等无状态的应用 

> 管理Pod和replicaSet

> 部署滚动升级等功能











