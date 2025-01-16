---
title: 웹소켓 비동기 처리 트러블슈팅
author: Changheon Lee
date: 2025-01-16
category: Java
layout: post
cover: https://changbill.github.io/assets/cover/localhost_issue.png
---

개인 프로젝트 중 생각보다 해결하는데 긴 시간이 소요된 버그가 있어 복습도 할겸 정리하려고 한다.

긴 시간이 걸린 이유는 버그의 원인을 전혀 다른 곳에서 찾고 있었기 때문이다.

그도 그럴것이 Spring Boot를 실행시켰는데 localhost가 연결이 안됐다.

![localhost_이슈](../assets/cover/localhost_issue.png)

처음 보는 버그였다.

인터넷에 검색을 해봐도 인텔리제이 실행을 잘못 눌러서 그렇다, 방화벽 문제다, 포트 포워딩을 잘못한거다 등등 현재의 버그와 다른 오류들만 보여줬기 때문에 스스로 해결해 가는수 밖에 없었다.

이전 commit까지는 잘 실행되었기 때문에 되돌아가본 결과 문제점을 찾아낼 수 있었다.

## 문제

메인 스레드에서 WebSocket 연결을 완료할 때까지 블로킹하여 localhost에서 연결을 거부하는 문제(Spring Boot 초기화 작업이 진행되지 않는 문제)가 발생한 것이다.

기존 코드는 다음과 같다.

```
@PostConstruct      // Spring이 빈 초기화를 완료한 후 자동으로 호출
public void startWebSocket() {
    try (HttpClient client = HttpClient.newHttpClient()) {
        webSocket = client.newWebSocketBuilder()
                .buildAsync(URI.create(BINANCE_WS_URL), new WebSocketListenerImpl())
                .join();
    }
}
```

문제가 발생한 주요 부분은 `@PostConstruct`와 `.join()`이다.

우선 `@PostConstruct`는 Spring에서 빈 초기화가 완료되면 호출하는 메서드 애노테이션이다.

Spring Boot의 초기화 과정에서, `@PostConstruct` 메서드가 완료되지 않으면 Spring은 해당 빈의 초기화가 아직 완료되지 않았다고 간주한다.

그리고 `.join()`은 CompletableFuture의 메서드로써 동기적으로 작동한다. WebSocket 연결이 완료될 때까지 호출 스레드를 블로킹한다. 현재 메인 스레드에서 호출하고 있으므로 메인 스레드가 블로킹되어 `@PostConstruct` 메서드를 완료하지 못하고 Spring Boot 초기화를 완료하지 못하는 것이다.

## 그렇다면 해결방법은?

동기적으로 작동하는 `.join()` 메서드를 비동기 메서드로 변경해주면 해결될 것 같다.

```
@PostConstruct
public void startWebSocket() {
    HttpClient client = HttpClient.newHttpClient();
    client.newWebSocketBuilder()
          .buildAsync(URI.create(BINANCE_WS_URL), new WebSocketListenerImpl())
          .thenAccept(webSocket -> this.webSocket = webSocket)
          .exceptionally(error -> {
              log.error("WebSocket 연결 실패: {}", error.getMessage());
              return null;
          });
}
```

검색해본 결과 `.thenAccept()`라는 메서드가 있었다. `.join()` 메서드는 CompletableFuture의 작업이 완료된 후, 동기적으로 결과(websocket)를 기다리는데 `.thenAccept()` 메서드는 비동기적으로 콜백 함수가 새로운 스레드에서 실행된다.

참고로 뒤에 `.exceptionally()` 메서드가 있는데 이는 예외 처리를 명시적으로 할 수 있도록 도와준다. 물론 없어도 `CompletionException`을 던진다.
