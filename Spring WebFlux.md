# Spring WebFlux
- 비동기 및 논블로킹(non-blocking) 방식으로 웹 애플리케이션을 개발할 수 있게 해주는 리액티브 프로그래밍 모델을 기반으로 하는 Spring Framework의 모듈

- Spring WebFlux는 특히 대규모의 동시성 처리가 필요한 웹 애플리케이션, 예를 들어 실시간 데이터 스트리밍이나 채팅 애플리케이션, 또는 고성능 API 서버 등에 적합

- Mono는 0 또는 1개의 데이터를 비동기적으로 처리하는 데 사용되고, Flux는 0부터 무한대의 데이터를 처리할 수 있는 스트림을 제공

``` java
@Autowired
private WebClient.Builder webClientBuilder;

public Mono<UserDto> getUserById(String userId) {
    return webClientBuilder.build()
        .get()
        .uri("http://user-service/users/{id}", userId)
        .retrieve()
        .bodyToMono(UserDto.class);
}
```

- 예시
``` java
    /**
     * 관리자 예약 신청 관리자의 경우 실시간이어도 이벤트 스트림 거치지 않고 바로 예약 처리
     *
     * @param saveRequestDto
     * @return
     */
    public Mono<ReserveResponseDto> create(ReserveSaveRequestDto saveRequestDto) {
        return Mono.just(saveRequestDto)
            .map(ReserveSaveRequestDto::createNewReserve)
            .zipWith(getUserId())
            .flatMap(tuple -> Mono.just(tuple.getT1().setCreatedInfo(LocalDateTime.now(), tuple.getT2())))
            .flatMap(validator::checkReserveItems)
            .onErrorResume(Mono::error)
            .flatMap(this::updateInventory)
            .onErrorResume(Mono::error)
            .flatMap(reserveRepository::insert)
            .flatMap(reserveRepository::loadRelations)
            .doOnNext(reserve -> sendAttachmentEntityInfo(streamBridge,
                AttachmentEntityMessage.builder()
                    .attachmentCode(reserve.getAttachmentCode())
                    .entityName(reserve.getClass().getName())
                    .entityId(reserve.getReserveId())
                    .build()))
            .flatMap(this::convertReserveResponseDto);


    }

```
이 코드는 **Reactive Programming** 스타일로 작성된 비동기적인 방식으로, **Mono**와 여러 **비동기 연산**을 체이닝하여 처리하는 흐름입니다. 기존 **동기 방식**에 익숙하다면 이 흐름이 조금 낯설 수 있지만, 단계별로 이해하면 그 구조를 이해하기 쉽습니다. 아래에서 각 연산이 무엇을 하는지 설명하겠습니다.

### 주요 흐름

1. **Mono.just(saveRequestDto)**:
   - 주어진 **`saveRequestDto`**를 **Mono**로 감싸 비동기 흐름을 시작합니다.

2. **`map(ReserveSaveRequestDto::createNewReserve)`**:
   - `saveRequestDto` 객체를 이용해 새로 예약 객체(예: `Reserve` 객체)를 생성합니다.
   - 이 단계는 **동기적 연산**이기 때문에 **map**을 사용하여 데이터를 변환합니다.

3. **`zipWith(getUserId())`**:
   - **`getUserId()`**는 아마도 비동기적으로 사용자 ID를 가져오는 메서드일 것입니다. 이 연산은 `Reserve` 객체와 **사용자 ID**를 결합하여 두 값을 묶은 **튜플(Tuple)**로 반환합니다.

4. **`flatMap(tuple -> Mono.just(tuple.getT1().setCreatedInfo(LocalDateTime.now(), tuple.getT2())))`**:
   - 묶인 **튜플**에서 `Reserve` 객체(`getT1()`)와 사용자 ID(`getT2()`)를 가져와, 해당 예약 정보에 **생성 정보를 설정**합니다.
   - **`flatMap`**을 사용하는 이유는, 이후 단계에서 **비동기 작업**을 처리할 준비를 하기 위함입니다.

5. **`flatMap(validator::checkReserveItems)`**:
   - **예약 항목을 검증**하는 비동기 작업을 수행합니다. 이 단계에서 항목에 문제가 있으면 검증 오류가 발생할 수 있습니다.

6. **`onErrorResume(Mono::error)`**:
   - 에러 발생 시, 에러를 다룰 수 있는 처리 로직을 추가합니다. 이 단계에서는 검증 오류나 기타 예외를 처리할 수 있습니다.

7. **`flatMap(this::updateInventory)`**:
   - 검증이 완료된 후, 재고를 업데이트하는 **비동기 작업**을 수행합니다. 여기서 `updateInventory()`는 재고 업데이트 관련 비동기 메서드로 추측됩니다.

8. **`flatMap(reserveRepository::insert)`**:
   - 재고 업데이트 후, **`reserveRepository.insert()`**를 호출하여 예약 정보를 데이터베이스에 삽입합니다.

9. **`flatMap(reserveRepository::loadRelations)`**:
   - 예약 정보가 삽입된 후, **관련된 데이터**를 로드하는 작업을 처리합니다. 예를 들어, 예약에 연결된 사용자 정보나 다른 연관 데이터를 가져올 수 있습니다.

10. **`doOnNext(reserve -> sendAttachmentEntityInfo(...))`**:
   - 예약이 성공적으로 삽입된 후, 예약과 관련된 **첨부 파일** 정보를 Kafka나 다른 메시징 시스템에 보내는 작업을 처리합니다. 이 단계는 비동기가 아니며, **사이드 이펙트(side effect)**로써 별도의 작업을 처리할 수 있습니다.
   - **`sendAttachmentEntityInfo()`**는 특정 첨부 파일 코드와 예약 ID 등을 기반으로 메시지를 발행하는 역할을 합니다.

11. **`flatMap(this::convertReserveResponseDto)`**:
   - 마지막으로, **`Reserve` 객체를 `ReserveResponseDto`로 변환**하는 비동기 작업을 수행하고, 최종적으로 이 변환된 DTO를 반환합니다.

### 전체 흐름 요약

1. **`saveRequestDto`로 새 `Reserve` 객체를 생성**하고, **사용자 ID**와 결합합니다.
2. 예약 생성 정보를 **검증**하고, **재고를 업데이트**합니다.
3. 예약 정보를 데이터베이스에 **삽입**하고, 관련된 데이터(연관된 엔티티나 관계 데이터)를 **로드**합니다.
4. 예약 완료 후, 첨부 파일 정보를 메시징 시스템으로 **전송**합니다.
5. 마지막으로 **`Reserve` 객체를 `ReserveResponseDto`로 변환**하고 반환합니다.

### 어려운 이유

이 코드가 기존의 동기 방식과 달라서 어려운 이유는, 모든 연산이 **비동기적으로** 처리되고 **Mono**로 데이터를 주고받는 방식 때문입니다. 기존 동기 방식에서는 결과가 즉시 반환되지만, 여기서는 각 단계가 **비동기**로 연결되며, **연산 체이닝**을 통해 작업의 흐름을 관리하고 있습니다.

- **동기 방식**에서는 데이터를 바로 반환하거나 처리하는 작업이 직관적입니다.
- **비동기 방식**에서는 데이터가 **Mono**나 **Flux**로 감싸져 있고, 이 데이터를 **단계별로 비동기적으로 처리**해야 하므로 흐름이 복잡해 보일 수 있습니다.

### 해결 방법

이 흐름을 이해하기 위해서는 **비동기 프로그래밍**의 기본 개념인 **Reactive Streams**나 **Mono/Flux**의 동작 방식을 먼저 이해하는 것이 좋습니다.

- **Mono**: 하나의 비동기 값 또는 오류를 나타냅니다.
- **flatMap**: 비동기 작업을 체이닝할 때 사용되며, 각 작업이 비동기적으로 처리될 수 있도록 합니다.
- **map**: 동기적으로 값을 변환할 때 사용됩니다.
- **zipWith**: 두 개의 비동기 작업을 결합하여 함께 처리할 때 사용합니다.

### 결론

이 코드는 **Reactive Programming**을 기반으로 작성된 비동기적 흐름을 처리하는 로직으로, **Mono**를 사용하여 여러 단계를 **비동기적으로 연결**하고 있습니다. 

- 동기코드 vs 비동기코드
    - 동기 코드
    ``` java
    @Override
    public OrderDto createOrder(OrderDto orderDto) {
        orderDto.setOrderId(UUID.randomUUID().toString());
        orderDto.setTotalPrice(orderDto.getQty() * orderDto.getUnitPrice());

        ModelMapper mapper = new ModelMapper();
        mapper.getConfiguration().setMatchingStrategy(MatchingStrategies.STRICT);
        OrderEntity orderEntity = mapper.map(orderDto, OrderEntity.class);

        orderRepository.save(orderEntity);

        OrderDto returnValue = mapper.map(orderEntity, OrderDto.class);

        return returnValue;
    }

    ```
    - 비동기 코드
    ``` java
    @Override
    public Mono<OrderDto> createOrder(OrderDto orderDto) {
        return Mono.just(orderDto)
            .map(dto -> {
                // DTO에서 기본값 설정
                dto.setOrderId(UUID.randomUUID().toString());
                dto.setTotalPrice(dto.getQty() * dto.getUnitPrice());
                return dto;
            })
            .map(dto -> modelMapper.map(dto, OrderEntity.class)) // DTO -> Entity 변환
            .flatMap(orderRepository::save) // 비동기적으로 엔티티 저장
            .map(entity -> modelMapper.map(entity, OrderDto.class)); // 저장 후 Entity -> DTO 변환
    }

    ```
<details>
<summary> MSA라고 반드시 WebFlux를 사용할 필요는 없다. </summary>

### MSA라고 반드시 WebFlux를 사용할 필요는 없다.

MSA로 전환하면서도 **모놀리식 코드를 대부분 유지**하면서 **각 서비스 간 데이터를 수정하거나 처리할 때 비동기 방식**(예: **Kafka**)을 사용하면 충분히 MSA의 이점을 활용할 수 있습니다. 꼭 **WebFlux**와 같은 **비동기 논블로킹** 프레임워크를 사용할 필요는 없습니다.

### 1. **모놀리식 코드 활용**
   - **MSA로 전환**할 때 기존의 모놀리식 코드를 그대로 사용하면서, **서비스 간의 통신**만 **비동기 방식**으로 처리하면 됩니다.
   - **동기 통신**(예: **RestTemplate**, **OpenFeign**)을 사용해도 문제없습니다. **즉각적인 응답**이 필요한 경우에는 **동기 방식**이 더 직관적이고 간단합니다.

### 2. **데이터 수정 시 비동기 처리**
   - 서비스 간에 **데이터를 수정**하거나 **주문 처리 후 재고를 업데이트**하는 등의 작업은 비동기 방식(**Kafka** 등)을 통해 처리하면 성능 및 확장성에서 이점을 얻을 수 있습니다.
   - **데이터 일관성**을 유지하면서 **서비스 간 독립성**을 높일 수 있기 때문에, 서비스 간 데이터 수정 시에는 비동기 처리 방식이 적합합니다.

### 3. **WebFlux를 꼭 사용할 필요는 없다**
   - **WebFlux**는 **비동기 논블로킹** 처리에 적합한 프레임워크이지만, 모든 상황에서 필요한 것은 아닙니다. 특히, 대부분의 시스템에서는 **동기 방식**이 더 직관적이고 관리하기 쉬운 경우가 많습니다.
   - **WebFlux**는 주로 **대규모 비동기 작업**을 처리하거나, **높은 동시성**이 필요한 경우에 사용됩니다. 그러나, 기본적으로 **RestTemplate** 또는 **OpenFeign**과 같은 **동기 방식**으로도 충분히 MSA의 요구를 충족할 수 있습니다.

### 4. **MSA로 전환 시의 단계적 접근**
   - **모놀리식 코드**를 사용하면서 **점진적으로 MSA**로 전환할 수 있습니다. 즉, 필요한 부분에서만 **서비스를 분리**하고, **비동기 처리**가 필요한 경우에만 **Kafka**와 같은 메시징 시스템을 도입하면 됩니다.
   - **웹 요청**이나 간단한 데이터 조회에는 **동기 방식**을 그대로 사용하고, **데이터 처리**나 **상태 변경** 작업에서만 **비동기 처리**를 도입하는 방식으로 설계할 수 있습니다.

### 예시 흐름

1. **주문 생성**:
   - **동기 방식**으로 **주문 서비스**에서 **사용자 정보**나 **상품 정보**를 조회한 후 주문을 처리.
   - 주문이 완료되면 **비동기 방식(Kafka)**으로 **재고 서비스**에 재고 감소 요청을 보냄.

2. **재고 업데이트**:
   - **재고 서비스**는 **Kafka 이벤트**를 받아 비동기적으로 **재고를 업데이트**함.

이 방식은 대부분의 모놀리식 코드를 유지하면서, 서비스 간 통신에서 **비동기 처리**를 도입하는 방식으로 MSA로 전환할 수 있는 효율적인 방법입니다.

### 결론

- **모놀리식 코드를 대부분 유지**하면서 **비동기 방식**으로 필요한 부분만 처리하면 충분히 MSA의 장점을 활용할 수 있습니다.
- **WebFlux**는 필수가 아니며, 필요에 따라 사용하면 되고, 대부분의 경우 **동기 방식**으로도 충분히 시스템을 관리할 수 있습니다.
- **Kafka**와 같은 비동기 시스템을 활용해 **서비스 간 데이터 수정** 및 **이벤트 처리**를 효율적으로 구현할 수 있습니다.
</details>