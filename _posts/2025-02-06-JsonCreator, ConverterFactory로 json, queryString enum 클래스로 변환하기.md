---
title: JsonCreator, ConverterFactory로 json, queryString enum 클래스로 변환하기
author: Changheon Lee
date: 2025-02-06
category: Java
layout: post
---

오늘은 json, queryString 요청을 JsonCreator(json) 어노테이션, ConverterFactory(queryString) 클래스를 통해 enum 클래스 자동변환을 해보도록 하자

```
@Getter
@RequiredArgsConstructor
public enum ProviderType{
    LOCAL("local"),
	GOOGLE("google"),
    KAKAO("kakao"),
    NAVER("naver");
}
```

위의 코드와 같이 json으로 데이터를 받을때 GOOGLE, KAKAO, NAVER로 받는것이 아닌 프론트 단에서 편리하도록 google, kakao, naver 또는 Google, Kakao, Naver 등의 형태로 받아야 할때가 있다 <br>

이때 JsonCreator 어노테이션과 ConverterFactory 클래스를 사용하면 보다 편하게 처리해줄 수 있다 <br>

## JsonCreator 어노테이션

JsonCreator 어노테이션을 사용하여 json데이터로 enum 타입이 넘어왔을 때 string을 enum으로 변환 처리 해줄 수 있다 <br>

```
@Getter
@RequiredArgsConstructor
public enum ProviderType {
    LOCAL("local"),
    GOOGLE("google"),
    KAKAO("kakao"),
    NAVER("naver");

    private final String registrationId;

    @JsonCreator(mode = JsonCreator.Mode.DELEGATING)
    public static ProviderType from(@JsonProperty("provider") String provider) {
        for (ProviderType type : ProviderType.values()) {
            if (type.name().equalsIgnoreCase(provider)) {
                return type;
            }
        }
        throw new ApplicationException(ErrorCode.INVALID_PROVIDER_TYPE);
    }
}
```

### @JsonCreator란

기본생성자 + setter 조합으로서 jackson이 해당 함수를 통해 객체를 생성하고 필드를 채워줌<br>

```
public record SocialUserJoinRequest(
        String email,
        ProviderType providerType,
        String password,
        String name,
        String nickname
) {
}
```

위의 코드와 같이 요청을 enum타입으로 받을 시 JsonCreator를 통해 받아온 String 타입을 enum타입으로 변환시켜주는 로직을 구현했다<br>

Request와 Response를 짜서 요청보내면

```
###GET http://localhost/enumtest
Content-Type : application/json

{
    "providerType": "google"
}
```

```
{
    "providerType": "GOOGLE"
}
```

잘 변환되어 나오는 것을 확인할 수 있다.

### ConverterFactory 클래스란

ConverterFactory는 query string(예: ?status=A)을 통해 전달된 문자열을 자동으로 Enum으로 변환 처리해 줄 수 있다<br>

우선 모든 Enum이 공통적으로 사용할 EnumMapperType 인터페이스를 만들어준다

```
public interface EnumMapperType {
    String getCode();
    String getRegistrationId();
}
```

그 다음 구현체로써 enum에 오버라이딩 한 뒤

```
@Getter
@RequiredArgsConstructor
public enum ProviderType implement EnumMapperType {
    LOCAL("local"),
    GOOGLE("google"),
    KAKAO("kakao"),
    NAVER("naver");

    private final String registrationId;

    @JsonCreator(mode = JsonCreator.Mode.DELEGATING)
    public static ProviderType from(@JsonProperty("provider") String provider) {
        for (ProviderType type : ProviderType.values()) {
            if (type.name().equalsIgnoreCase(provider)) {
                return type;
            }
        }
        throw new ApplicationException(ErrorCode.INVALID_PROVIDER_TYPE);
    }

    @Override
    public String getCode() {
        return name();
    }
}
```

ConverterFactory를 사용하여 문자열을 Enum으로 변환하는 로직을 작성한다

```
public class StringToEnumConverterFactory implements ConverterFactory<String, Enum<? extends EnumMapperType>> {
    @Override
    public <T extends Enum<? extends EnumMapperType>> Converter<String, T> getConverter(Class<T> targetType) {
        return new ProviderTypeConverter<>(targetType);
    }

    private static final class ProviderTypeConverter<T extends Enum<? extends EnumMapperType>> implements Converter<String, T> {

        private final Map<String, T> map;

        public ProviderTypeConverter(Class<T> targetEnum) {
            map = Arrays.stream(targetEnum.getEnumConstants())
                    .collect(Collectors.toMap(enumConstant -> ((EnumMapperType) enumConstant).getRegistrationId(), Function.identity()));
        }

        @Override
        public T convert(String source) {
            if (!StringUtils.hasText(source)) {
                return null;
            }

            T value = map.get(source.toLowerCase());
            if (value == null) {
                throw new IllegalArgumentException("유효하지 않은 ProviderType: " + source);
            }
            return value;
        }
    }
}
```

- `ConverterFactory<String, Enum<? extends EnumMapperType>>`
  → String을 받아 Enum으로 변환하는 역할

- `ProviderTypeConverter<T>`
  → targetEnum.getEnumConstants()를 통해 모든 Enum 값을 가져와서 Map<String, T>로 변환
  → source 값과 매칭되는 Enum을 찾아 반환

Spring MVC에서 FormatterRegistry에 변환기를 등록하면 자동 변환이 적용된다

```
@Configuration
public class WebConfiguration implements WebMvcConfigurer {
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverterFactory(new StringToEnumConverterFactory());
    }
}
```

```
@RestController
@RequestMapping("/auth")
public class AuthController {

    @GetMapping
    public String getProvider(@RequestParam ProviderType provider) {
        return "입력된 Provider: " + provider;
    }
}
```

결과

```
GET /auth?provider=google	입력된 Provider: GOOGLE
GET /auth?provider=kakao	입력된 Provider: KAKAO
GET /auth?provider=naver	입력된 Provider: NAVER
GET /auth?provider=xyz	400 Bad Request (예외 발생)
```
