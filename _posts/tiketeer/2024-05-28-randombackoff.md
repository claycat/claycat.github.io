---
title: "랜덤 백오프를 사용한 낙관적 락 성능개선"
date: 2024-05-28 12:00:00 +/-TTTT
categories: [Tiketeer]
comments: true
tags: [java, tiketeer]
---

# 서론

<img src="/images/../../assets/images/2024-06-15-20-13-33.png" alt="Description" style="display:block; width:400px; margin-left:auto; margin-right:auto;">

낙관적 락을 사용하여 티켓팅 시스템을 구현하는 과정에서

적용한 성능개선 방안 중 하나인 랜덤 백오프 입니다.

낙관적 락에 대한 정보 및 배경은 [링크](https://www.notion.so/4975208574964df8beaefc6a753ad837?pvs=21)에서 찾아보실 수 있습니다.

낙관적 락을 효과적으로 사용하기 위해서는 애플리케이션에서 개발자가 직접 예외가 발생하는것을 잡아주고,

재시도를 수행해야 합니다.

이번 아티클에서는 재시도 방식에 무작위성을 추가하여 얻어지는 성능 개선을 확인해보겠습니다.

# 본론

[기존 구현방식](https://www.notion.so/4975208574964df8beaefc6a753ad837?pvs=21)에서는 동시성 문제가 발생할 경우 실패한 모든 클라이언트들은 즉시 재시도합니다.

<img src="/images/../../assets/images/2024-06-15-20-14-08.png" alt="Description" style="display:block; width:500px; margin-left:auto; margin-right:auto;">

이 방식으로 구현할 경우, 모든 클라이언트는 동일한 간격으로 재시도하기 때문에 부하가 특정 시점에 집중됩니다.

방지하기 위해서는 클라이언트가 실패했을 때 매번 동일한 간격으로 재시도하는것이 아닌, 불규칙적인 간격을 두고 재시도해야 합니다.

이 “불규칙적인 간격”을 Jitter라고 합니다.

(재시도 방식에는 Jitter를 제외하고도 Exponential Backoff, Exponential Jitter Backoff 등 다양한 방법이 있지만, 여기서는 Linear Jitter를 위주로 설명하겠습니다)

기존 코드에서 어떻게 재시도 간격에 불규칙성을 추가할 수 있는지 알아보도록 하겠습니다.

<img src="/images/../../assets/images/2024-06-15-20-14-14.png" alt="Description" style="display:block; width:700px; margin-left:auto; margin-right:auto;">

`@Retryable` 애노테이션은 인자로 backoff 값을 받으며, delayExpression을 SpeL (Spring Expression Language)를 사용하여 값을 조절할 수 있습니다.

> delayExpression : delay값을 결정할 표현식
> maxDelay : 최대 delay 값
> multiplier : delay가 증가할 배율 (exponential backoff 등에 용이)

위 식을 사용할 경우, delay값은 매번 300 - 500 사이의 임의의 값으로 결정됩니다.

## 측정

<img src="/images/../../assets/images/2024-06-15-20-14-35.png" alt="Description" style="display:block; width:1000px; margin-left:auto; margin-right:auto;">

좌: 랜덤, 우: 고정

랜덤 간격과 고정 간격의 성능을 비교해보기 위해서 위처럼 코드를 작성하고 부하 테스트를 진행해보았습니다.

K6를 이용하였으며, 부하 스크립트는 [링크](https://github.com/Tiketeer/Tiketeer-BE/blob/develop/stress-test/k6-scripts/stress.js)에서 확인할 수 있습니다.

총 500명의 유저와 50개의 티켓이 있는 환경에서 측정한 결과, 아래와 같은 실험 결과를 얻을 수 있었습니다.

<img src="/images/../../assets/images/2024-06-15-20-14-43.png" alt="Description" style="display:block; width:1200px; margin-left:auto; margin-right:auto;">

랜덤 백오프를 적용한 결과 평균 요청 시간은 약 760ms 였으며,

고정 백오프를 사용한 결과의 평균 요청 시간은 약 942ms 였습니다.

간단하게 개선 결과를 연산해보면 아래와 같습니다.

$$\text{Percentage Change} = \left( \frac{\text{New Value} - \text{Old Value}}{\text{Old Value}} \right) \times 100$$

$$
\text{Percentage Change} = \left( \frac{962 \, \text{ms} - 760 \, \text{ms}}{760 \, \text{ms}} \right) \times 100


$$

$$
\text{Percentage Change} = \left( \frac{202 \, \text{ms}}{760 \, \text{ms}} \right) \times 100 \approx 26.58\%


$$

약 26%의 개선을 얻을 수 있었습니다.

## 부록 - 동적 재시도 간격

간혹 재시도 로직이 필요할 때, 동적으로 재시도 범위를 설정해야 하는 경우가 있습니다.

저희 팀 같은 경우, [재시도 범위를 어떻게 설정해야 할까?](https://www.notion.so/K6-72add37df17448c4bbf91585867deada?pvs=21) 라는 실험을 진행할 때 동적인 재시도 범위 설정이 필요했는데요,

템플릿 문자열처럼

`"#{T(java.util.concurrent.ThreadLocalRandom).current().nextInt(${min}, ${max})}"`

쉽게 작성이 가능하다면 좋겠지만, 아쉽게도 SpeL에서는 불가능합니다.

동적으로 재시도 간격을 설정하기 위해서는 애노테이션이 아닌 `RetryTemplate`를 사용해야 합니다.

<img src="/images/../../assets/images/2024-06-15-20-18-16.png" alt="Description" style="display:block; width:800px; margin-left:auto; margin-right:auto;">

`RetryTemplate`은 편리하게도 `UniformRandomBackoffPolicy()`라는 랜덤 간격을 제공하는 구현체가 존재합니다.

해당 구현체를 이용해 최소값과 최댓값을 설정해주고, `builder`를 이용해 `RetryTemplate`을 작성하면 사용이 가능합니다.

해당 코드는 [링크](https://github.com/Tiketeer/Tiketeer-LockTest/blob/develop/src/main/java/com/tiketeer/Tiketeer/domain/purchase/usecase/CreatePurchaseOLockUseCase.java)에서 찾아보실 수 있습니다.

# 결론

이번 문서에서는 다수의 클라이언트가 동시성 문제를 발생시키는 환경에서 성능을 최적화할 수 있는 **랜덤 백오프**에 대해서 알아보았습니다.

대기열 큐가 구현되어있는 환경에서는 동시에 요청하는 유저의 수가 제한되기 때문에 재시도 전략에 따른 성능차이가 미미할 수 있지만,

다수의 클라이언트가 동일한 간격으로 재시도를 요청하는 경우에는 간단하게 재시도 간격에 무작위성을 추가하는것을 통해 성능개선을 이룰 수 있었습니다.

감사합니다.
