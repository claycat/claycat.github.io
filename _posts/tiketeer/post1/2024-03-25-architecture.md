---
title: 스프링 코드 아키텍처
date: 2024-03-25 12:00:00 +/-TTTT
categories: [Tiketeer]
comments: true
tags: [spring, tiketeer]
---

작성자 : [Claycat](https://github.com/claycat)

## 개요

### 여러분들의 백엔드 프로젝트 아키텍처는 어떤가요?

이번에 지인들과 함께 Java/스프링 기반의 사이드 프로젝트를 진행하고 있습니다.  
프로젝트를 진행하던 중 초기 아키텍처 및 디렉토리 구조에 대해서 많은 고민이 있었는데요.  
결정 과정에 대해서 소개해 보려고 합니다.

해당 프로젝트의 Github [여기](https://github.com/Tiketeer/Tiketeer-BE) 입니다.

## 서론

이전에 진행했던 프로젝트들은 전통적인 Layered Architecture로 이루어졌습니다.
![asdf](/assets/img/posts/tiketeer/architecture/layered3.png)

일반적으로 Controller, Service, Repository 세개의 레이어로 구성되어있습니다.
Controller에서는 Service에 의존하고, Service에서는 Repository에 의존하는 방식입니다.

조금 더 구체적인 예시를 들면 다음과 같습니다.
![alt text](assets/img/posts/tiketeer/architecture/layered3_2.png)

해당 방식은 소규모의 프로젝트에서는 문제가 없었지만,  
다양한 유틸리티 메소드와 비즈니스 로직이 많아지고, API 엔드포인트들이 많아질수록 서비스단의 코드가 비대해졌습니다.  
서비스 코드가 몇백줄을 넘어가면서 유지보수성은 떨어지고 가독성 또한 낮아졌습니다.

## 원인

이 문제의 원인은 컨트롤러 - 서비스간의 1:1 관계 때문이라고 분석했습니다.
위 Member 예제처럼 MemberController - MemberService로 연관되어있는 경우입니다.  
이 경우, MemberService는 MemberController의 모든 엔드포인트를 메소드로 구현해야 합니다.  
Member에 대한 모든 비즈니스 로직을 포함할 수 밖에 없고, 거대한 테스트 코드는 덤입니다.

저희는 해당 도메인을 단 하나의 서비스 객체로 나타내는 구조는 거대한 객체를 유발하고,  
가독성과 유지보수성이 떨어지며,  
객체지향적으로도 너무 많은 책임을 갖는 좋지 않은 구조라고 판단하였습니다.

## 대안

해당 문제를 어떻게 해결할 수 있을지 회의를 하였고, 몇가지 제안들이 있었습니다.

1. CrudService

   - ✋ 비대한 서비스 코드를 줄일 수 있도록 CRUD관련 엔드포인트들만 별도로 처리하는 서비스 객체를 사용하자!

2. Hexagonal Architecture & Clean Architecture

   - ✋ 익숙한 계층 아키텍처 대신 새로운 아키텍처를 도입해서 사용하자!

3. DDD의 아이디어 채용 - UseCase에 중점을 두자

   - ✋ 꼭 하나의 서비스가 여러개의 엔드포인트를 처리해야 할까?  
     하나의 엔드포인트에 대한 하나의 서비스(유즈케이스)로서 대응하는건 어떨까?

## 논의

1. CrudService

   - 🙆 비대한 서비스 코드의 부담을 일부분 덜어줄 수 있다는 것은 동의.
     - 🙅 하지만 결국 본질적인 문제를 해결하는것은 아니다.
     - 🙅 CRUD이외의 엔드포인트가 다수 추가된다면 결국 비대해지는것은 마찬가지

2. Hexagonal & Clean Architecture

   - 기존 계층구조로 구성된 코드를 모두 뒤엎어야 함
     - 🙅 참고할 수 있는 정석적인 자료가 부재하며, 참고자료마다 구현방식이 모두 다름
   - 🙆 Ports & Adapter를 통해 외부 입력과 출력에 대해서 독립성을 확보할 수 있다
     - 🙅 in/out에 대한 port 및 adapter를 모두 작성해야 하는 불편함이 존재
     - 🙅 사이드 프로젝트의 규모상 웹 요청방식은 HTTP이외가 추가되지 않을 것이며,  
       마찬가지로 영속성 툴 또한 메인 DB (MySQL)에서 추가되지 않을 것인데, 오버엔지니어링이 아닐까

3. UseCase 중심
   - 🙆 익숙한 편이며, 서비스를 잘게 쪼갤 수 있다.
   - 🙅 결국 모든 엔드포인트에 대해서 대응하는 유즈케이스 객체가 만들어질텐데, 너무 많아지지 않을까?
     - 🙆 어차피 작업중인 유즈케이스만 보게 될 것
     - 🙆 동시에 여러 유즈케이스를 보는 상황이 온다면 의존성 문제가 생긴게 아닐까

## 결론

논의 결과, 저희는 UseCase를 중심으로 한, 세분화된 서비스 객체의 계층구조로 결정하였습니다.

결론은 다음과 같습니다.

- 컨트롤러는 기존과 동일한 역할을 수행하고, 도메인별로 분리합니다.

  - API 엔드포인트 상
    - /members/\*\* -> MemberController
    - /ticketings/\*\* -> TicketingController

- 기존 서비스를 메소드별로 UseCase로 나누어 별도의 객체로 분리합니다.
  - MemberService 하위의
    - memberService.login -> LoginUseCase
    - memberService.register -> RegisterUseCase
    - memberService.logout -> LogoutUseCase

코드를 살펴보면 다음과 같습니다.

```java
//편의상 주입 생략

@RestController
@RequestMapping("/members")
public class MemberController {

	@PostMapping("/register")
	public ApiResponse<MemberRegisterResponseDto> registerMember(
		final @Valid @RequestBody MemberRegisterRequestDto registerMemberDto) {
		return ApiResponse.wrap(
			MemberRegisterResponseDto.toDto(memberRegisterUseCase.register(MemberRegisterCommandDto.builder()
				.email(registerMemberDto.getEmail())
				.isSeller(registerMemberDto.getIsSeller())
				.build()
			))
		);
	}

	@PostMapping(path = "/password-reset/mail")
	public ResponseEntity sendPasswordChangeEmail() {
		var email = securityContextHelper.getEmailInToken();
		sendPasswordChangeEmailUseCase.sendEmail(
			SendPwdChangeEmailCommandDto.builder().email(email).build());
		return ResponseEntity.ok().build();
	}

	@PostMapping(path = "/{memberId}/points")
	public ResponseEntity<ApiResponse<ChargePointResponseDto>> chargePoint(@PathVariable String memberId,
		@Valid @RequestBody ChargePointRequestDto request) {
		var email = securityContextHelper.getEmailInToken();
		var totalPoint = chargeMemberPointUseCase.chargePoint(request.convertToCommandDto(memberId, email))
			.getTotalPoint();
		var result = ChargePointResponseDto.builder().totalPoint(totalPoint).build();
		return ResponseEntity.status(HttpStatus.OK).body(ApiResponse.wrap(result));
	}
}
```

## 재사용성과 의존성

해당 방식을 논의하던 중, 여러 유즈케이스에서 공통적으로 의존하는 로직이나 모듈에 대한 지적이 있었습니다.  
공통적인 로직을 처리할 때는 \*\*\*Service라는 네이밍으로 적절한 추상화와 객체지향적 센스를 발휘하여 사용하기로 정하였습니다.

예시를 살펴보면 다음과 같습니다.

구매자의 "구매목록"을 담당하는 Purchase 도메인 내부의 생성 및 삭제 유즈 케이스입니다.

```java
@Service
public class CreatePurchaseUseCase {

	@Transactional
	public CreatePurchaseResultDto createPurchase(CreatePurchaseCommandDto command) {

        //PurchaseService
		purchaseService.validateTicketingSalePeriod(ticketingId, command.getCommandCreatedAt());

		var newPurchase = purchaseRepository.save(Purchase.builder().member(member).build());
		var tickets = ticketRepository.findByTicketingIdAndPurchaseIsNullOrderById(
			ticketingId, Limit.of(count));

		if (tickets.size() < count) {
			throw new NotEnoughTicketException();
		}

		tickets.forEach(ticket -> {
			ticket.setPurchase(newPurchase);
		});

		return CreatePurchaseResultDto.builder()
			.purchaseId(newPurchase.getId())
			.createdAt(newPurchase.getCreatedAt())
			.build();
	}
}


@Service
public class DeletePurchaseTicketsUseCase {

	@Transactional
	public void deletePurchaseTickets(DeletePurchaseTicketsCommandDto command) {
		var purchase = purchaseRepository.findById(command.getPurchaseId()).orElseThrow(
			PurchaseNotFoundException::new);
		var ticketsUnderPurchase = purchaseService.findTicketsUnderPurchase(purchase.getId());
		var ticketsToRefund = ticketRepository.findAllById(command.getTicketIds());
		var ticketing = ticketsUnderPurchase.getFirst().getTicketing();

        //PurchaseService
		purchaseService.validatePurchaseOwnership(purchase.getId(), command.getMemberEmail());
		purchaseService.validateTicketingSalePeriod(ticketing.getId(), command.getCommandCreatedAt());

		var ticketIdUnderPurchase = ticketsUnderPurchase.stream().map(Ticket::getId).toList();
		AtomicInteger numOfDeletedTicket = new AtomicInteger();
		ticketsToRefund.forEach(ticket -> {
			if (ticketIdUnderPurchase.contains(ticket.getId())) {
				ticket.setPurchase(null);
				numOfDeletedTicket.getAndIncrement();
			}
		});
		if (numOfDeletedTicket.get() == ticketsUnderPurchase.size()) {
			purchaseRepository.delete(purchase);
		}
	}
}

```

두가지 유즈케이스 모두 "구매"에 대한 Validation을 하는 공통 로직에 의존하고 있기 때문에  
해당 로직은 PurchaseService에 위임하였습니다.

다이어그램으로 살펴보면 다음과 같습니다.
![alt text](assets/img/posts/tiketeer/architecture/1.png)

이를 통해 공통 로직과 의존도를 분리할 수 있었습니다.

## 후기

개인적으로 더이상 거대한 Service 객체를 보지 않아도 되는 면에서 가독성만큼은 훨씬 낫다고 생각합니다.
하나의 유즈케이스에 대해서만 집중해도 되니 유틸리티 메소드에 대한 의존성 관리도 쾌적해졌다고 느꼈습니다.  
이전에 사용하던 계층 구조와 선택하라고 한다면 분명 현재를 고를 것입니다.  
다만 우려되거나 아쉬운 부분 또한 존재합니다.

특히 우려되는 점은 서비스에 대한 오염입니다.  
적절한 책임의 분리를 하지 않고 모든 유틸리티 메소드들을 밀어넣는 용도로 ㅁㅁㅁService를 사용한다면  
결국 재사용성이 떨어지는 거대객체가 될 확률이 높습니다.

이 글을 작성하면서 보니, 위 언급된 이미지에서의 PurchaseService 또한 Validation과 Find의 유틸리티 책임이 혼합되어있습니다.  
PurchaseValidationService와 PurchaseFindService로 쪼개는게 더 적절할 수도 있습니다.

아쉬운 부분은 Hexagonal 과 Clean Architecture, Domain Driven Design등에 대해서 조금 더 깊은 이해를 했거나  
경험이 있었다면 더 좋은 아키텍처가 있을 수도 있다는 것이었습니다.  
해당 부분은 사이드 프로젝트를 마무리한다면 깊게 한번 다시 탐구해보도록 하겠습니다.

감사합니다.
