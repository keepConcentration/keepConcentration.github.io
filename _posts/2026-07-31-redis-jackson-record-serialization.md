---
layout: post
title: "Spring Data Redis와 Jackson: Record와 Stream.toList()의 직렬화 함정 해결하기"
date: 2026-07-31 10:00:00 +0900
categories: [Java, Redis]
tags: [Jackson, Redis, SpringDataRedis, Record, 직렬화, Stream.toList, 캐시]
---

회사에서 Redis 캐시를 붙이다가 Jackson 직렬화에서 한 번 크게 걸렸다. 파고들다 보니 [Spring Data Redis #2697](https://github.com/spring-projects/spring-data-redis/issues/2697)까지 갔고, 거기서 내가 쓴 해결법을 댓글로 남겼다.

시간이 지나서 우연히 본 건데, 모르는 오픈소스 프로젝트([goods-wms #37](https://github.com/2026-KB-WMS/goods-wms/pull/37))가 그 이슈를 인용하면서 같은 문제를 풀고 있더라. 내 댓글이 누군가의 설계 근거가 된 거다.

그래서 Record랑 `Stream.toList()`가 어떻게 직렬화에서 충돌하는지, 어떻게 풀었는지, 그리고 "기여는 코드만이 아니다"는 생각을 정리해본다.

## Redis 캐시가 갑자기 깨진다

`@Cacheable`로 조회 결과를 Redis에 넣고 있었다. 근데 특정 API에서만 역직렬화가 터졌다.

```java
@Cacheable("inventories")
public List<InventoryResult> getInventories(Long warehouseId) {
    return inventoryRepository.findByWarehouseId(warehouseId)
        .stream()
        .map(InventoryResult::from)
        .toList();  // 여기서 문제 발생
}
```

에러는 이런 식이었다.

```
InvalidTypeIdException: Could not resolve type id '2953' as a subtype of `java.lang.Object`
```

처음엔 "왜 리스트 첫 번째 요소가 타입 ID로 읽히지?" 싶었다. 알고 보니 Jackson Default Typing이 `ImmutableCollections$ListN`을 만났을 때 나오는 알려진 함정이었다.

## 원인: 세 가지가 한꺼번에 겹쳤다

### 1. Record는 암묵적으로 final이다

Java Record는 컴파일러가 알아서 `final` 클래스로 만든다. Jackson의 `DefaultTyping.NON_FINAL`은 `useForType()`에서 final을 만나면 타입 메타데이터를 빼버린다.

```java
// Jackson DefaultTyping.NON_FINAL 내부 로직 (의사코드)
public boolean useForType(JavaType t) {
    // final 클래스면 무조건 타입 정보 생략. Record도 final이니까 같이 걸린다.
    // Jackson은 isRecord()를 따로 체크하지 않는다.
    if (t.isFinal()) return false;
    return true;
}
```

그래서 `GenericJackson2JsonRedisSerializer`가 `readValue(bytes, Object.class)`로 읽을 때 타입 정보가 없고, 결국 `ClassCastException`이 난다.

### 2. `Stream.toList()`는 불변 내부 구현체를 돌려준다

`Stream.toList()` 문서는 "unmodifiable list"라고만 한다. 실제로는 JDK 내부 클래스인 `java.util.ImmutableCollections$List12` / `$ListN`을 반환한다.

이 구현체가 문제를 만든다.

첫째, NON_FINAL 전략에서는 타입 정보가 빠진다. `ImmutableCollections$ListN`은 `final`이고 `java.util` 패키지에 있지만 public이 아니라서 Jackson이 타입 정보를 생략한다. JSON이 `["java.util.ArrayList", [...]]`가 아니라 `[2953]`처럼 나가버린다. 역직렬화할 때 Jackson이 첫 요소 `2953`을 타입 ID로 읽으려다 실패한다.

둘째, EVERYTHING으로 타입을 억지로 넣어도 실패한다. `ImmutableCollections$List12`가 JSON에 찍히긴 하는데, public 생성자가 없어서 Jackson이 인스턴스를 못 만든다.

전략이랑 버전마다 깨지는 모습이 다르긴 하다. 그래도 구현 타입이 보장되지 않는 불변 컬렉션을 캐시 직렬화 포맷에 그대로 넣는 것 자체가 위험하다.

### 3. PROPERTY와 WRAPPER_ARRAY가 섞여 있다

`JsonTypeInfo.As.PROPERTY`는 이름만 보면 `{"@class": "...", "id": 1}` 형태만 쓸 것 같다. 근데 Jackson 내부에서는 배열/리스트 타입에 PROPERTY를 골라도 WRAPPER_ARRAY 규칙으로 우회한다.

`GenericJackson2JsonRedisSerializer`가 root collection을 다룰 때 이렇게 갈린다.

- Root가 객체 형태(`{"@class": ...}`)면 `TypeResolver.resolveType()`이 미리 타입을 잡는다.
- Root가 배열 형태(`[...]`)면 `Object.class` 기반 untyped 경로로 떨어지고, 이때 `AsPropertyTypeDeserializer`가 `_arrayDelegate()`(= `AsArrayTypeDeserializer`)에 위임하면서 `MismatchedInputException`이 난다.

```
MismatchedInputException: Unexpected token (START_OBJECT)
  AsArrayTypeDeserializer._locateTypeId
  AsArrayTypeDeserializer._deserialize
  AsArrayTypeDeserializer.deserializeTypedFromArray
  AsPropertyTypeDeserializer.deserializeTypedFromAny   ← _arrayDelegate()에 위임
  UntypedObjectDeserializerNR.deserializeWithType
```

## 해결: RecordSupportingTypeResolver + WRAPPER_ARRAY

### 1. Record용 커스텀 TypeResolver

`DefaultTypeResolverBuilder`를 상속해서 `useForType()`만 오버라이드했다. Record일 때는 NON_FINAL의 final 제외 로직을 건너뛴다.

```java
public class RecordSupportingTypeResolver extends DefaultTypeResolverBuilder {

    public RecordSupportingTypeResolver(DefaultTyping t, PolymorphicTypeValidator ptv) {
        super(t, ptv);
    }

    @Override
    public boolean useForType(JavaType type) {
        // Record는 final이어도 타입 정보 포함
        if (type.getRawClass().isRecord()) {
            return true;
        }
        return super.useForType(type);
    }
}
```

### 2. PolymorphicTypeValidator 범위 제한

```java
BasicPolymorphicTypeValidator.builder()
    .allowIfSubType("com.example")  // 프로젝트 도메인
    .allowIfSubType("java.util")                // ArrayList 등 컬렉션
    .build()
```

`allowIfSubType(Object.class)`에 EVERYTHING을 같이 쓰면 역직렬화 때 임의 클래스 인스턴스화 공격에 열린다. 그래서 패키지 단위로 막아뒀다.

### 3. WRAPPER_ARRAY 포맷

PROPERTY는 Jackson 2.18에서 `UntypedObjectDeserializerNR` 동작이 바뀌면서 `readValue(json, Object.class)` 역직렬화가 깨지는 이슈가 있다. 그래서 WRAPPER_ARRAY를 골랐다.

```java
// WRAPPER_ARRAY: ["타입ID", {필드들}]
objectMapper.activateDefaultTyping(
    ptv,
    ObjectMapper.DefaultTyping.NON_FINAL,
    JsonTypeInfo.As.WRAPPER_ARRAY
);
```

### 4. `toList()` → `collect(Collectors.toList())`

`Stream.toList()`를 `collect(Collectors.toList())`로 바꿨다. Jackson 편의만이 아니다. 구현 타입을 보장하지 않는 값을 캐시 포맷처럼 쓰지 않으려고 바꾼 거다.

- `Stream.toList()` → `java.util.ImmutableCollections$ListN` (private, 생성자 없음)
- `Collectors.toList()` → `java.util.ArrayList` (public 기본 생성자 있음)

결과 JSON은 `["java.util.ArrayList", [...]]` 형태로 남고, 직렬화/역직렬화 왕복이 된다.

### 덤으로 따라온 것: Redis 메모리 30% 절감

이 과정에서 부수 효과가 하나 있었다. 기존에는 `EVERYTHING` 전략으로 모든 객체에 타입 메타데이터를 붙이고 있었는데, 패키지 기반 `PolymorphicTypeValidator`로 범위를 제한하고 `WRAPPER_ARRAY` 포맷으로 전환하면서 Redis에 저장되는 문자열 길이가 크게 줄었다. 결과적으로 Redis 메모리 사용량이 약 30% 절감됐다.

## Spring Data Redis #2697에서의 이야기

이 문제는 Spring Data Redis 이슈에서도 꽤 길게 논의됐다. [spring-projects/spring-data-redis#2697](https://github.com/spring-projects/spring-data-redis/issues/2697)에서 `mp911de`는 이렇게 설명했다.

> "The difference between the two values is that the `ArrayList` variant is non-final while `Stream.toList()` produces a final and private type `ImmutableCollections$ListN`. The typing mechanism skips adding type hints if the type is `final` and originates from the `java.` namespace..."

나도 그 스레드에 재현 케이스랑 분석을 올렸다. `jxblum`이 추가한 테스트는 이런 식이다.

```java
@Test // GH-2697
void serializingDeserializingIntegerListIsHandledCorrectly() {
    GenericJackson2JsonRedisSerializer redisSerializer =
        new GenericJackson2JsonRedisSerializer();

    List<Integer> integers = Stream.of(2953).toList();
    // ...
    redisSerializer.deserialize(redisSerializer.serialize(integers));
}
```

이슈는 아직 Open이고, Spring 팀도 `for: team-attention` 라벨을 달아둔 상태다.

## 우연한 발견: 댓글이 다른 프로젝트에 닿았다

시간이 지나서 [goods-wms #37](https://github.com/2026-KB-WMS/goods-wms/pull/37) PR을 발견했다. 나와는 상관없는 프로젝트인데, PR 본문에 spring-data-redis #2697을 관련 이슈로 적어두고 있었다.

> "문제 2. `Stream.toList()`, `List.of()`: 불변 내부 구현체가 캐시에 노출"
>
> > 관련 이슈: [spring-data-redis #2697](https://github.com/spring-projects/spring-data-redis/issues/2697)

그 PR 작성자는 #2697 논의를 읽고 같은 문제를 자기 쪽에서 푼 거였다. `RecordSupportingTypeResolver` 설계 근거로도 그 이슈가 나왔다.

댓글만 남겼는데도 다른 사람한테 도움이 됐다는 게 신기하고 좀 뿌듯했다.

## 마치며: 기여는 코드만이 아니다

기술적으로 배운 건 이거다. Jackson Default Typing은 Java 16+의 불변 타입들과 잘 안 맞는다. `Stream.toList()`, `List.of()`, `Arrays.asList()`는 JDK 내부 불변 구현체를 반환하고, 이건 public API가 아니라서 Jackson 같은 라이브러리에서 예측이 안 된다.

캐시 직렬화 포맷으로 쓸 때는 `ArrayList`, `LinkedList`처럼 구현체가 보장된 컬렉션을 쓰거나, Jackson 타입 해석을 제대로 이해하고 커스터마이징해야 한다.

기술 밖에서도 느낀 게 있다.

나는 Spring Data Redis #2697에 코드 PR을 올린 적이 없다. 재현이랑 원인 분석만 댓글로 남겼을 뿐이다. 근데 시간이 지나 그 댓글이 다른 개발자 문제 해결에 참고 자료로 인용되는 걸 봤다. 모르는 프로젝트 PR에서 그 스레드가 설계 근거로 나왔다.

오픈소스 기여라고 하면 코드 PR, 머지된 커밋만 떠올리기 쉽다. 근데 이슈에 재현을 정리하고, 원인을 적고, 삽질한 경험을 공유하는 것도 기여다. 그런 기록이 있어야 다음 사람이 같은 함정에 덜 빠진다.

앞으로도 삽질한 내용은 이슈든 블로그든 꾸준히 남겨야겠다.

참고 자료:

- [Spring Data Redis #2697. GenericJackson2JsonRedisSerializer can't deserialize previously serialized Stream.toList()](https://github.com/spring-projects/spring-data-redis/issues/2697)
- [goods-wms #37. Redis 직렬화 전략을 EVERYTHING에서 RecordSupportingTypeResolver와 WRAPPER_ARRAY로 교체](https://github.com/2026-KB-WMS/goods-wms/pull/37)
- [jackson-databind #3404](https://github.com/FasterXML/jackson-databind/issues/3404)
- [jackson-databind #3344](https://github.com/FasterXML/jackson-databind/issues/3344)
- [jackson-databind #4849](https://github.com/FasterXML/jackson-databind/issues/4849)
- [jackson-databind #5223](https://github.com/FasterXML/jackson-databind/issues/5223)
