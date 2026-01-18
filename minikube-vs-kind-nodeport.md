# 미니큐브 vs kind: NodePort 비교

이 문서는 NodePort 접근 방식의 차이를 비교합니다.

## 차이점 설명

- 미니큐브는 `minikube service`가 NodePort 접근을 쉽게 보조해줍니다.
- kind는 노드가 컨테이너라서 NodePort만으로는 호스트 접근이 어려울 수 있습니다.
- `extraPortMappings` 설정이나 포트포워딩을 함께 고려해야 합니다.

```mermaid

flowchart LR
  subgraph M[미니큐브 NodePort]
    m1[NodePort Service]
    m2[minikube service로 접근 보조]
    m3[로컬 포트 매핑이 쉬움]
    m1 --> m2 --> m3
  end

  subgraph K[kind NodePort]
    k1[NodePort Service]
    k2[노드 컨테이너 포트 노출 필요]
    k3[extraPortMappings 설정 권장]
    k1 --> k2 --> k3
  end

  m3 -.차이점.-> k3
```
