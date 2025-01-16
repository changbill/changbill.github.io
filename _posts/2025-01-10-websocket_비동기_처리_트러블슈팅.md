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
    CompletableFuture.runAsync(this::initializeWebSocket);
}

private void initializeWebSocket() {
    try {
        HttpClient client = HttpClient.newHttpClient();
        webSocket = client.newWebSocketBuilder()
                .buildAsync(URI.create(BINANCE_WS_URL), new WebSocketListenerImpl())
                .join();
    } catch (Exception e) {
        log.error("WebSocket 초기화 실패: {}", e.getMessage());
    }
}
```

알아본 결과 `.runAsync()`라는 메서드가 있었다. `.join()` 메서드는 CompletableFuture의 작업이 완료된 후, 동기적으로 결과(websocket)를 기다리는데 `.runAsync()` 메서드는 매개변수 Supplier 함수가 비동기적으로 새로운 스레드에서 실행된다.

## 이번 기회에 CompletableFuture에 대해 정리해보자

### 비동기 작업 시작

1. supplyAsync()
   supplyAsync() 메서드는 Supplier 함수형 인터페이스를 인자로 받아, 해당 함수형 인터페이스의 get() 메서드가 실행하는 작업을 비동기적으로 수행합니다. 이 메서드는 작업의 결과를 CompletableFuture 객체로 감싸서 반환하는데, 이 CompletableFuture 객체는 비동기 작업의 결과를 나타내며 작업의 상태 정보를 담고 있다.

아래 코드는 supplyAsync()를 사용한 예

```
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    // 비동기적으로 실행할 작업
    String result = "Hello, World!";
    return result;
});
```

2. runAsync()
   runAsync() 메서드는 Runnable 인터페이스를 인자로 받아, 해당 인터페이스의 run() 메서드가 실행하는 작업을 비동기적으로 수행합니다. 이 메서드는 작업의 결과를 반환하지 않으므로, 반환 타입은 CompletableFuture<Void>가 됩니다.

아래 코드는 runAsync()를 사용한 예입니다.

CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
// 비동기적으로 실행할 작업
System.out.println("Hello, World!");
});
이처럼, supplyAsync()와 runAsync() 메서드는 비동기 작업을 수행하는 방법을 제공하지만, 작업의 결과를 반환하는지 여부에 따라 사용하는 인터페이스와 반환 타입이 다릅니다.

정리

비동기 결과 조작
thenApply[Async], thenAccept[Async], thenRun[Async]은 CompletableFuture 클래스에서 연속적인 작업을 처리하는 방법들입니다.

1. thenApply / thenApplyAsync
   thenApply[Async] 메서드는 모두 이전 작업의 결과를 받아 새로운 연산을 수행하고 그 결과를 CompletableFuture로 반환하는 기능을 제공합니다.

CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
// 비동기적으로 실행할 작업
return "Hello";
}).thenApply(result -> {
// 이전 작업의 결과를 받아서 다른 연산 수행
return result + ", World!";
});

결과: "Hello World!"
이 코드를 실행하면 "Hello, World!"라는 결과를 얻을 수 있습니다. 여기서 thenApply 메서드는 이전 작업이 완료된 후에 함수를 동일한 스레드에서 실행하며, 이전 작업과 동기적으로 연산이 이루어집니다.

반면, thenApplyAsync 메서드는 이전 작업이 완료된 후에 함수를 새로운 스레드 또는 스레드 풀에서 비동기적으로 실행합니다. 이 방식은 이전 작업과 독립적으로 연산이 이루어지며, 작업의 병렬 처리를 가능하게 합니다.

CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
// 비동기적으로 실행할 작업
return "Hello";
}).thenApplyAsync(result -> {
// 이전 작업의 결과를 받아서 다른 연산 수행
return result + ", World!";
});
이 코드를 실행하면 마찬가지로 "Hello, World!"라는 결과를 얻을 수 있습니다. 이처럼 thenApplyAsync를 사용하면, 비동기 작업의 결과를 기반으로 새로운 작업을 수행하면서도 메인 스레드가 블로킹되는 것을 방지할 수 있습니다.

CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
// 비동기적으로 실행할 작업
return "Hello";
}).thenApplyAsync(result -> {
// 이전 작업의 결과를 받아서 다른 연산 수행
return result + ", World!";
});

System.out.println("Continuing with other work...");
위의 코드에서는 thenApplyAsync가 이전 작업의 결과를 기다리는 동안에도 메인 스레드는 "Continuing with other work..."라는 문장을 출력하는 등의 다른 작업을 계속해서 수행할 수 있습니다.

결론적으로, thenApply와 thenApplyAsync는 모두 이전 작업의 결과를 기반으로 새로운 작업을 수행하지만, 동작 방식에는 차이가 있습니다. thenApply는 동기적으로 같은 스레드에서 작업이 이루어지는 반면, thenApplyAsync는 비동기적으로 별도의 스레드에서 작업이 수행됩니다.

2. thenAccept / thenAcceptAsync
   이전 단계의 결과를 받아 해당 결과를 소비하는 작업을 수행하고, 반환값이 없습니다. 아래 코드는 supplyAsync로 비동기 작업을 시작하고, 그 결과를 받아서 thenAccept로 소비하는 예입니다.

CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> {
// 비동기적으로 실행할 작업
return "Hello, World!";
}).thenAccept(result -> {
// 이전 작업의 결과를 받아서 소비만 함
System.out.println(result);
});

결과: "Hello World!"
실행하면 "Hello World!"라는 결과를 출력하며, 반환값이 없으므로 CompletableFuture<Void>가 생성됩니다. thenAcceptAsync 메서드는 마찬가지로 thenAccept와 동일한 작업을 수행하지만, 별도의 스레드에서 비동기적으로 연산을 수행합니다.

3. thenRun / thenRunAsync
   이전 단계의 결과와 상관없이 특정 작업을 실행합니다. 아래 코드는 supplyAsync로 비동기 작업을 시작하고, 그 작업이 끝나면 결과와 상관없이 "Finished!"라는 문자열을 출력하는 예입니다.

CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> {
// 비동기적으로 실행할 작업
return "Hello, World!";
}).thenRun(() -> {
// 이전 작업의 결과와 상관없이 실행할 작업
System.out.println("Finished!");
});

결과: "Finished!"
이를 실행하면 "Finished!"라는 결과를 출력하며, 반환값이 없으므로 CompletableFuture<Void>가 생성됩니다. thenRunAsync 메서드는 thenRun 메서드와 같은 작업을 수행하지만, 별도의 스레드에서 비동기적으로 연산을 수행합니다.

정리

비동기 작업 조합
비동기 작업을 조합하는 방법에는 thenCompose[Async], thenCombine[Async], allOf(), anyOf() 등의 메서드가 있습니다.

1. thenCompose / thenComposeAsync
   thenCompose()와 thenComposeAsync() 메서드는 이전 작업의 결과를 이용하여 새로운 CompletableFuture를 생성하고 실행하는 데 사용됩니다. 이 메서드들은 이전 작업의 결과를 Function의 인자로 받아, 새로운 CompletableFuture를 반환하는 함수를 실행합니다.

다음은 thenCompose()를 사용한 예입니다.

CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
// 비동기적으로 실행할 작업
return "Hello";
}).thenCompose(result -> {
// 이전 작업의 결과를 받아 새로운 CompletableFuture를 생성하고 실행
return CompletableFuture.supplyAsync(() -> {
return result + ", World!";
});
});
위 코드에서 thenCompose() 메서드는 이전 작업의 결과인 "Hello" 문자열을 인자로 받아, 새로운 CompletableFuture를 생성하고 실행합니다. 이 새로운 CompletableFuture는 이전 작업의 결과에 ", World!"를 추가하여 새로운 문자열을 반환합니다.

thenComposeAsync() 메소드는 thenCompose()와 비슷하지만, 새로운 CompletableFuture를 생성하고 실행하는 함수를 비동기적으로 실행합니다. 이 메소드는 이전 작업과 독립적으로 새로운 작업을 수행할 수 있어, 작업의 병렬 처리를 가능하게 합니다.

다음은 thenComposeAsync()를 사용한 예입니다.

CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
// 비동기적으로 실행할 작업
return "Hello";
}).thenComposeAsync(result -> {
// 이전 작업의 결과를 받아 새로운 CompletableFuture를 생성하고 실행
return CompletableFuture.supplyAsync(() -> {
return result + ", World!";
});
});
위 코드에서 thenComposeAsync() 메소드는 이전 작업의 결과인 "Hello" 문자열을 인자로 받아, 새로운 CompletableFuture를 생성하고 실행하는 함수를 비동기적으로 실행합니다. 그리고 이 새로운 CompletableFuture는 이전 작업의 결과에 ", World!"를 추가하여 새로운 문자열을 반환합니다.

따라서 thenCompose()와 thenComposeAsync() 메소드는 이전 작업의 결과를 기반으로 새로운 비동기 작업을 생성하고 실행하는 데 사용됩니다. 이 둘의 차이점은 새로운 비동기 작업을 동기적으로 실행하느냐(thenCompose()) 아니면 비동기적으로 실행하느냐(thenComposeAsync())에 있습니다.

thenCompose[Async] vs thenApply[Async]
그리고 여기서 비동기 작업 조합인 thenCompose[Async]와 비동기 결과 조작인 thenApply[Async]가 비슷하다고 느낄 수도 있는데, 주요 차이점은 다음과 같습니다.

thenApply()는 이전 작업의 결과를 받아 특정 연산을 수행하고 그 결과를 반환하는 반면, thenCompose[Async]는 이전 작업의 결과를 활용하여 새로운 비동기 작업을 생성하고 그 결과를 반환합니다.
즉, thenApply()는 주로 이전 작업의 결과를 변환하는 데 사용되며, thenCompose[Async]는 이전 작업의 결과를 바탕으로 새로운 비동기 작업을 시작하는 데 사용됩니다.

2. thenCombine / thenCombineAsync
   thenCombine[Async] 메서드는 두 개의 CompletableFuture가 모두 완료될 때까지 기다린 후, 두 작업의 결과를 이용하여 새로운 값을 계산합니다. thenCombine[Async]는 두 작업의 결과를 BiFunction의 인자로 받아, 새로운 값을 계산하는 함수를 실행합니다.

다음은 thenCombine()를 사용한 예입니다.

CompletableFuture<String> hello = CompletableFuture.supplyAsync(() -> {
return "Hello";
});

CompletableFuture<String> world = CompletableFuture.supplyAsync(() -> {
return "World";
});

CompletableFuture<String> future = hello.thenCombine(world, (h, w) -> {
return h + ", " + w + "!";
});
위 코드에서 thenCombine() 메소드는 두 개의 CompletableFuture, hello와 world가 모두 완료될 때까지 기다립니다. 두 작업이 모두 완료되면, thenCombine() 메소드는 두 작업의 결과인 "Hello"와 "World"를 인자로 받아, 새로운 값을 계산하는 함수를 실행합니다. 이 새로운 값은 "Hello, World!"입니다.

지금까지와 마찬가지로 thenCombineAsync() 메소드는 thenCombine()와 비슷하지만, 새로운 값을 계산하는 함수를 비동기적으로 실행하며, 두 작업과 독립적으로 새로운 값을 계산할 수 있어, 작업의 병렬 처리를 가능하게 합니다.

thenCompose[Async] vs thenCombine[Async]
여기서 thenCompose[Async]와 thenCombine[Async]의 주요 차이점은 다음과 같습니다.

thenCompose[Async]는 하나의 CompletableFuture의 결과를 이용하여 새로운 CompletableFuture를 생성하고 실행합니다.
즉, thenCompose[Async]는 체인 형태의 비동기 작업을 처리하는 데 사용되며, thenCombine[Async]는 두 개의 독립적인 비동기 작업을 병렬로 처리하고 그 결과를 합치는 데 사용됩니다.

3. allOf()
   allOf()메서드는 여러 개의 CompletableFuture를 배열로 받아, 모든 비동기 작업이 동시에 실행되도록 합니다. 또한 모든 작업이 완료될 때까지 기다렸다가, 모든 작업이 완료되면 CompletableFuture<Void>를 반환합니다. 이를 통해 모든 비동기 작업의 완료를 알릴 수 있습니다.

CompletableFuture<String> hello = CompletableFuture.supplyAsync(() -> {
return "Hello";
});

CompletableFuture<String> world = CompletableFuture.supplyAsync(() -> {
return "World";
});

CompletableFuture<Void> future = CompletableFuture.allOf(hello, world);
위 코드에서 allOf() 메서드는 hello와 world 두 CompletableFuture를 배열로 받아, 두 작업이 모두 완료될 때까지 기다립니다. 두 작업이 모두 완료되면, allOf() 메소드는 CompletableFuture<Void>를 반환하여 모든 작업의 완료를 알립니다.

allOf() 메소드는 여러 비동기 작업이 동시에 수행되어야 하고, 모든 작업이 완료될 때까지 기다려야 하는 상황에서 유용하게 사용될 수 있습니다. 이런 특성 덕분에 allOf() 메소드는 주로 병렬 처리 작업에 사용되는데, 여러 비동기 작업을 동시에 수행하고 모든 작업이 완료되는 시점을 알아야 할 때 allOf() 메서드를 활용하면 효율적인 비동기 프로그래밍이 가능합니다.

4. anyOf()
   anyOf() 메서드는 여러 개의 CompletableFuture 중에서 가장 먼저 완료되는 작업의 결과를 반환합니다. 또한 CompletableFuture<Object>를 반환하며, 가장 먼저 완료되는 작업의 결과를 알리는 데 사용됩니다.

다음은 anyOf()를 사용한 예입니다.

CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
return "cf1";
});

CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> {
return "cf2";
});

CompletableFuture<Object> future = CompletableFuture.anyOf(cf1, cf2);
위 코드에서 anyOf() 메서드는 cf1과 cf2 두 개의 CompletableFuture 중에서 가장 먼저 완료되는 작업의 결과를 반환합니다. 두 작업 중 어느 것이 먼저 완료되더라도, anyOf() 메서드는 CompletableFuture<Object>를 반환하여 가장 먼저 완료되는 작업의 결과를 알립니다.

anyOf() 메서드는 여러 비동기 작업 중에서 가장 빠르게 완료되는 작업의 결과를 얻고자 할 때 유용하게 사용됩니다. 여러 비동기 작업을 동시에 수행하고, 그중 가장 빠르게 완료되는 결과만 필요한 상황에서 anyOf() 메서드를 활용하면 효율적인 비동기 프로그래밍이 가능합니다.

정리

비동기 예외처리

1. handle[Async]
   handle[Async] 메서드는 비동기 작업의 결과를 처리하거나, 작업 중 발생한 예외를 처리하는 역할을 합니다. 작업이 정상적으로 완료되면 결괏값을 반환하고, 예외가 발생하면 예외를 처리합니다. handle[Async]은 결과값 또는 예외를 인자로 받아 처리한 후, 새로운 값을 반환합니다.

CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
if (true) {
throw new RuntimeException("Exception!");
}
return 100;
}).handle((res, ex) -> {
if (ex != null) {
System.out.println("예외 발생: " + ex);
return -1;
}
return res;
});
이 코드에서 CompletableFuture.supplyAsync() 메서드를 통해 비동기 작업을 시작합니다. 이 작업은 예외를 발생시키므로 handle 메서드에는 예외 객체가 전달됩니다. handle 메서드는 예외를 처리하고, 예외가 발생했으므로 -1을 반환합니다.

2. whenComlete[Async]
   whenComplete[Async] 메서드는 비동기 작업의 결과나 예외를 받아 처리하되, 새로운 값을 반환하지 않습니다. 원래의 CompletableFuture에 영향을 미치지 않고 예외를 처리하는 역할만 합니다.

CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
if (true) {
throw new RuntimeException("Exception!");
}
return 100;
}).whenComplete((res, ex) -> {
if (ex != null) {
System.out.println("예외 발생: " + ex);
}
});
이 코드에서 CompletableFuture.supplyAsync() 메서드는 비동기 작업을 시작하며, 이 작업은 예외를 발생시킵니다. whenComplete 메서드는 예외가 발생했을 때 이를 처리하지만 새로운 값을 반환하지 않으므로, 원래의 CompletableFuture는 그대로 유지됩니다.

따라서 whenComplete[Async] 메서드를 사용하면 비동기 작업의 결과를 처리하거나, 작업 중 발생한 예외를 적절히 처리할 수 있으며, 이때 원래의 CompletableFuture에는 영향을 미치지 않습니다.

3. exceptionally[Async]
   exceptionally[Async] 메서드는 비동기 작업 중 예외가 발생했을 때 대체 값을 반환하는 역할을 합니다. 예외를 인자로 받아 처리하며, 예외가 발생했을 때 반환할 대체 값을 지정합니다.

CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
if (true) {
throw new RuntimeException("Exception!");
}
return 100;
}).exceptionally(ex -> {
System.out.println("예외 발생: " + ex);
return -1;
});
이 코드에서 CompletableFuture.supplyAsync() 메서드는 비동기 작업을 시작하며, 이 작업은 예외를 발생시킵니다. exceptionally() 메서드는 예외가 발생했을 때 이를 처리하고, 예외가 발생했으므로 -1을 반환합니다.

따라서 exceptionally[Async] 메서드를 사용하면 비동기 작업 중 예외가 발생했을 때 적절한 대체 값을 반환할 수 있습니다.

정리

비동기 대기 / 취소 처리

1. get()
   get() 메서드는 비동기 작업의 결과를 반환하며, 작업이 완료될 때까지 호출한 스레드를 대기 상태로 만듭니다. 블로킹 방식으로 동작하므로 get() 메서드를 호출한 스레드는 작업의 완료를 기다리는 동안 다른 작업을 진행하지 않습니다.

또한, get() 메서드는 InterruptedException과 ExecutionException 등의 체크된 예외(Checked Exception)를 던집니다. 따라서 get() 메서드를 사용할 때는 try-catch 블록을 통해 이러한 예외를 적절히 처리해야 합니다.

아래는 get() 메서드를 사용하는 예제 코드입니다.

CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "Hello");
try {
String result = future.get(); // 작업이 완료될 때까지 대기
System.out.println(result); // "Hello" 출력
} catch (InterruptedException | ExecutionException e) {
e.printStackTrace();
}

2. get(timeout, unit)
   get(timeout, unit) 메서드는 비동기 작업의 결과를 반환하며, 작업의 완료를 기다리는 시간을 제한할 수 있습니다. get() 메서드와 유사하지만, 작업 완료를 기다리는 시간을 제한하는 점이 다릅니다.

만약 지정된 시간 동안 작업이 완료되지 않으면 TimeoutException이 발생합니다. 여기서 시간 단위는 TimeUnit 열거형을 사용하여 지정합니다. 또한 get(timeout, unit)은 작업 중 예외가 발생하면 ExecutionException을 던지므로, 이를 처리하기 위해 try-catch 블록을 사용해야 합니다.

아래는 get(timeout, unit) 메서드를 사용하는 예제 코드입니다.

CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
try {
Thread.sleep(2000); // 2초 대기
return "Hello";
} catch (InterruptedException e) {
throw new IllegalStateException(e);
}
});
try {
String result = future.get(1, TimeUnit.SECONDS); // 1초 동안만 대기
System.out.println(result);
} catch (InterruptedException | ExecutionException | TimeoutException e) {
e.printStackTrace();
}

3. join()
   join() 메서드는 비동기 작업의 결과를 반환하며, 작업이 완료될 때까지 현재 스레드를 대기 상태로 만듭니다. get() 메서드와 유사한 역할을 하지만 join() 메서드는 체크된 예외가 아닌 CompletionException이라는 언체크 된 예외를 발생시킵니다.

또한, join() 메서드는 블로킹 메서드로 작용하여, 해당 메서드가 호출된 스레드는 비동기 작업이 완료될 때까지 다른 작업을 수행하지 않고 대기합니다.

아래는 join() 메서드를 사용하는 예제 코드입니다.

CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
return "Hello";
});

// join() 메서드로 비동기 작업의 결과를 가져옵니다.
// 이 메서드는 작업이 완료될 때까지 현재 스레드를 블로킹합니다.
String result = future.join();

System.out.println(result);
이 코드에서 join() 메서드를 통해 비동기 작업의 결과를 얻을 수 있습니다. 또한 작업이 완료될 때까지 스레드를 블로킹하기 때문에, 작업이 완료되면 그 결과를 반환하고, 그렇지 않으면 스레드를 계속 대기 상태로 유지합니다.

4. cancel(boolean mayInterruptIfrunning)
   cancel(boolean mayInterruptIfRunning) 메서드는 비동기 작업을 취소하는 데 사용되며, 매개변수 mayInterruptIfRunning은 작업이 실행 중일 때 그 작업을 중단시킬지 여부를 결정합니다. 만약 이 매개변수가 true로 설정되면, 실행 중인 작업이 있다면 그 작업을 중단시킵니다.

작업이 성공적으로 취소되면 true를 반환하며 작업이 이미 완료되었거나 이미 취소되었거나, 취소할 수 없는 경우에는 false를 반환합니다.

아래는 cancel(boolean mayInterruptIfRunning) 메서드를 사용하는 예제 코드입니다.

CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
try {
Thread.sleep(2000); // 2초 동안 대기
return "Hello";
} catch (InterruptedException e) {
throw new IllegalStateException(e);
}
});

// cancel 메서드를 호출하여 비동기 작업을 취소합니다.
// 이 메서드는 작업이 성공적으로 취소되면 true를 반환합니다.
boolean isCancelled = future.cancel(true);

// 작업 취소 여부를 출력합니다.
// 이 예제에서는 true가 출력됩니다.
System.out.println(isCancelled);
이 코드에서 cancel(true) 부분은 비동기 작업을 취소하며, 실행 중인 작업이 있다면 그 작업을 중단시키는 것을 의미합니다. 이 작업이 성공적으로 취소되면 true를 반환하고, 그렇지 않으면 false를 반환합니다.

정리
