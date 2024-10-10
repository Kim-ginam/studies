# OpenFeign vs RestTemplate
- OpenFeign
    - OpenFeign은 마이크로서비스 아키텍처에서 다른 서비스와의 HTTP 통신을 매우 쉽게 만들 수 있는 도구
    - Spring Cloud 환경에서 많이 사용되며, 마이크로서비스 간의 효율적인 통신을 지원
    - OpenFeign은 다른 서비스의 API를 호출하는 클라이언트 역할을 수행
    - @RestController는 클라이언트의 요청을 처리하는 서버 역할을 수행
    - 각 마이크로서비스를 알고 있을 때 사용하기 적합
    - 직관적이고, 자바 호출처럼 사용이 가능
```java
@FeignClient(name="catalog-service", configuration = FeignErrorDecoder2.class)
public interface CatalogServiceClient {

    @GetMapping("/catalog-service/getCatalogs_wrong")
    List<ResponseCatalog> getCatalogs();

}
```
- RestTemplate
    - 환경 설정에 정의된 IP를 기반으로 URL과 HTTP 메서드를 사용하여 데이터를 전송하고, getBody를 통해 응답 데이터를 받습니다.
    - 전통적으로 많이 사용된 방식입니다.
    
```java
    @Override
    public String getAroundHub() {
        //uri는 어떤경로로 요청을 할건지해서 사용하는 클래스
        URI uri= UriComponentsBuilder
                .fromUriString("httpL//localhost:8081")
                .path("/api/server/around-hub")
                .encode() //기본적 utf-8로
                .build() //return 값 uricomponent 이기때문에
                .toUri(); //uri로 바꿔줌

        RestTemplate restTemplate=new RestTemplate();
        // 응답을 ResponseEntity 객체로 받는다. getForObject()와 달리 HTTP 응답에대한
        // 추가 정보를 담고 있어서 GET 요청에 대한 응답 코드, 실제 데이터를 확인할 수 있다.
        // 또한 ResponseEntity<T> 제네릭 타입에 따라서 응답을 String이나 Object 객체로 받을 수 있다.
        ResponseEntity<String> responseEntity=restTemplate.getForEntity(uri, String.class); //만들어진 uri값과 그 타입과 맞춘 클래스

        LOGGER.info("status code: {}", responseEntity.getStatusCode());
        LOGGER.info("body : {}", responseEntity.getBody());

        return responseEntity.getBody();
    }
 - restTemplate.getForEntity(serverUrl, String.class)는 지정된 URL로 GET 요청을 보내고, 응답을 ResponseEntity 형태로 받습니다.
 - getBody() 메서드를 통해 실제 응답 데이터를 얻을 수 있습니다.
```

- OpenFeign의 장점
    - 로그를 통해 호출 과정을 볼 수 있으며, FeignException을 통해 에러 발생 시 해당 부분만 제외하고 나머지 작업을 계속 진행할 수 있습니다.

## ErrorDecoder
- ErrorDecoder를 통해 Feign 클라이언트에서 발생한 에러를 상태 코드 값에 따라 분기 처리할 수 있습니다.
```java
@Bean
public FeignErrorDecoder getFeignErrorDecoder() {
    return new FeignErrorDecoder();
}

@Override
public Exception decode(String methodKey, Response response) {
    switch(response.status()) {
        case 400:
            break;
        case 404:
            if (methodKey.contains("getOrders")) {
                return new ResponseStatusException(HttpStatus.valueOf(response.status()),
                       "User's orders is empty");
            }
            break;
        default:
            return new Exception(response.reason());
    }

    return null;
}
 - 환경 변수(env)를 사용하여 에러 메시지를 출력할 수 있습니다.
 - Config 정보가 외부에 있을 경우, 이를 변경하고 Spring Cloud Bus를 통해 리프레시 할 수 있습니다.
```