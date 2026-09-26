---
title: "GPU Traversal Prefetch and Latency Hiding"
date: "2026-09-26"
category: Graphics
tags: [GPU, Rendering, Prefetch]
level: intermediate
---

# [Daily Graphics Growth] 2026-09-26 - GPU Traversal Prefetch and Latency Hiding

## 1. 오늘의 개념
GPU hierarchy traversal의 future-node prefetch와 latency hiding을 다룬다.

## 2. 한 줄 핵심
Traversal state를 future-address hint로 활용한다.

## 3. 왜 중요한가
Memory-bound traversal의 stall을 줄일 수 있다.

## 4. 구현 관점
Stack, frontier, treelet에서 미래 node를 찾아 software pipeline으로 겹친다.

## 5. 내 관심 분야와 연결
Dynamic SDF와 Vulkan/CUDA traversal에 연결된다.

## 6. 머릿속에 남길 질문 3개
1. Prefetch distance는 어떻게 정할까?
2. Register pressure와 latency hiding은 어떻게 trade-off할까?
3. Metadata와 payload prefetch를 어떻게 분리할까?

## 7. graphics engineer 면접 질문 1개와 답변
질문: Prefetch만 하면 빨라지는가?
답변: Address lead time, independent compute, occupancy가 함께 필요하다.

## 8. 포트폴리오 / 커리어 연결
GPU memory hierarchy와 traversal scheduling을 함께 설명할 수 있다.

## 9. 내일 이어서 볼 개념
GPU Traversal Continuations.

## 10. 참고 키워드
GPU Prefetch, cp.async, TTP, BVH, Treelet, Vulkan Subgroup.
