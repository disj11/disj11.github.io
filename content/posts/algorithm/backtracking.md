---
title: Backtracking (되추적 기법)
date: 2019-06-08T20:55:00+09:00
menu:
  sidebar:
    name: Backtracking (되추적 기법)
    identifier: backtracking
    parent: algorithm
    weight: 10
---

## 개요

되추적 기법은 DFS (깊이 우선 탐색)를 이용하여 트리를 순회하며 해를 찾는다. 이때에 중요한 것은, 자식 노드로 갈 땐 항상 해가 있을 수 있는지 없는지 확인을 해 본다는 것이다. 확인한 결과 더 이상 탐색할
필요가 없다면 현재 방향으로는 나아가는 것을 멈추고, 다른 자식 노드로 방향을 바꾼다. 아래의 문제를 통해 자세히 알아보자.

## 저울 문제

양팔 저울과 1번부터 n 번까지 n 개의 추가 있고, 각 추 i의 무게를 W(i)라고 할 때, 무게 M인 물체가 주어지면 이를 양팔 저울로 달 수 있는지 판정하는 문제이다.   
`입력 : 추의 개수 n, 물체의 무게 M, 추의 무게 W[1...n]`   
`출력 : 무게 M을 달 수 있는지 (true, false)`

```
// 입력
4
13
7 8 5 3

// 출력
true
```

## 원리

저울 문제의 상태 공간 트리를 표현 하기 위한 조건을 생각해보면, 아래 두 가지 조건을 생각 할 수 있다.

1. w(i) 번째 추를 선택하는 경우
2. w(i) 번째 추를 선택하지 않는 경우

위 조건을 생각해 냈다면, 일단 DFS 를 사용하여 추를 선택하는 방향으로 탐색을 시작한다. 그러다 더 이상 자식 노드의 방향으로 진행하여도 해가 나오지 않는 경우, 추를 선택하지 않는 방향으로 바꾸면 된다. 여기서
자식 노드의 방향으로 진행하여도 해가 나오지 않는 경우란 아래와 같다.

1. 현재까지 선택한 추들의 무게 합이 m보다 클 경우
2. 현재까지 선택한 추들의 무게 합과 앞으로 선택할 수 있는 추들의 무게 합이 m보다 작은 경우

이러한 경우를 잘 생각하며 아래의 코드를 확인하면 Backtracking 이 무엇인지 어느정도 이해할 수 있을 것이다.

```java
import java.util.Scanner;

public class Balances {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        int n = scanner.nextInt(); // 추의 개수
        int m = scanner.nextInt(); // 물체의 무게
        int[] w = new int[n]; // 추의 무게

        int ssum = 0; // 현재까지 선택한 추들의 무게 합
        int rsum = 0; // 앞으로 선택할 수 있는 추들의 무게 합

        for (int i = 0; i < n; i++) {
            w[i] = scanner.nextInt();
            rsum += w[i];
        }

        System.out.println(test(0, ssum, rsum, w, n, m));
    }

    /**
     * 주어진 추를 이용하여 무게를 잴 수 있는지 판단하는 메서드
     * @param level 현재 노드의 레벨 (추의 번호)
     * @param ssum 현재 노드까지 선택된 추의 무게 합
     * @param rsum 남은 추들의 무게 합
     * @param w 추의 무게
     * @param n 추의 개수
     * @param m 물체의 무게
     * @return 물체를 잴 수 있는지 여부
     */
    private static boolean test(int level, int ssum, int rsum, int[] w, int n, int m) {
        // 왼쪽 자식 탐색 (level + 1 번째 추를 선택 했을 경우)
        int nextWeight = ssum + w[level + 1];
        if (nextWeight <= m && (ssum + rsum) >= m) {
            if (nextWeight == m) {
                return true;
            } else if (level < n - 1 && test(level + 1, nextWeight, rsum - w[level + 1], w, n, m)) {
                return true;
            }
        }

        // 오른쪽 자식 탐색 (level + 1 번째 추를 선택하지 않을 경우)
        if (ssum <= m && (ssum + rsum - w[level + 1]) >= m) {
            if (ssum == m) {
                return true;
            } else if (level < n - 1 && test(level + 1, ssum, rsum - w[level + 1], w, n, m)) {
                return true;
            }
        }

        return false;
    }
}
```

## 성능 분석

각 노드에서의 처리 시간은 상수이므로 시간 복잡도는 탐색하는 노드의 수에 비례한다. 상태 공간 트리는 높이가 (n + 1)인 포화 이진 트리이므로 전체 노드의 수는 2<sup>n+1</sup> - 1 이다. 즉,
되추적이 발생하지 않는 최악의 경우 시간 복잡도는 O(2<sup>n</sup>)이다. 최악의 시간 복잡도는 지수 시간이지만, 대부분의 경우는 이 보다 빠른 시간 안에 종료된다. 하지만 이를 보장하지는 못 한다.