---
title: SSE (Server-Sent Events)
date: 2019-04-19T16:30:00+09:00
menu:
  sidebar:
    name: SSE (Server-Sent Events)
    identifier: server-sent-events
    parent: spring
    weight: 10
---

## 개요

실시간으로 웹 애플리게이션 서버에서 클라이언트로 데이터를 전송하는 방법 중 하나이다. 이러한 방법에는 WebSoket을 이용하는 방법도 있지만, WebSocket은 서버와 클라이언트의 양방향 통신을 하기 위해
사용한다면, SSE는 서버에서 클라이언트로 데이터를 전송하는 단방향 통신만 필요로 하는 경우 사용한다.

## 특징

1. HTML5 표준
2. 전통적인 HTTP를 사용하여 통신함
3. 연결이 끊어졌을 경우 재접속 시도 하는 등 대부분의 저수준 처리가 되어있음

## 브라우저 지원

SSE는 HTML5 표준 기술로 명시되어있으며, 대부분의 브라우저에서 지원한다.

[브라우저 호환성](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events#browser_compatibility)

## 참고 사항

1. 클라이언트에서 연결된 이후 30초(default)간 서버에서 이벤트를 받지 못하면 연결이 끊어진다.
2. 연결이 끊어질 경우 클라이언트는 재연결을 시도한다.
3. 연결 유지 시간은 `application.yaml` 파일의 `spring.mvc.async.request-timeout`로 변경 할 수 있다.

## 간단한 예제

```java
import org.springframework.stereotype.Service;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

import java.io.IOException;
import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.UUID;

@Service
public class NotificationService {
    private final DateFormat DATE_FORMATTER = new SimpleDateFormat("dd-MM-yyyy hh:mm:ss a");
    private final List<SseEmitter> emitters = new ArrayList<>();

    public SseEmitter subscribe() {
        SseEmitter emitter = new SseEmitter();

        emitter.onCompletion(() -> emitters.remove(emitter));
        emitter.onTimeout(() -> emitters.remove(emitter));
        emitters.add(emitter);

        postMessage();
        return emitter;
    }

    private void postMessage() {
        List<SseEmitter> deadEmitters = new ArrayList<>();
        emitters.forEach(emitter -> {
            try {
                // SseEmitter.event().name("eventName").data(...)
                // 이런식으로 이벤트명을 정의 할 수 있음
                emitter.send(SseEmitter.event()
                        .data(DATE_FORMATTER.format(new Date()) + " : " + UUID.randomUUID().toString()));
            } catch (IOException e) {
                deadEmitters.add(emitter);
            }
        });
        emitters.removeAll(deadEmitters);
    }
}
```

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.context.request.async.AsyncRequestTimeoutException;
import org.springframework.web.servlet.mvc.method.annotation.ResponseBodyEmitter;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

@RestController
@RequestMapping("/notification")
public class NotificationController {
    private final NotificationService notificationService;

    public NotificationController(NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    @GetMapping(produces = "text/event-stream")
    public ResponseEntity<ResponseBodyEmitter> doNotify() {
        final SseEmitter emitter = notificationService.subscribe();
        return new ResponseEntity<>(emitter, HttpStatus.OK);
    }

    // timeout 마다 에러 로그 찍는 것 방지
    @ExceptionHandler(value = AsyncRequestTimeoutException.class)
    public String asyncTimeout(AsyncRequestTimeoutException e) {
        return "SSE timeout... OK";
    }
}
```

```javascript
const eventSource = new EventSource('/notification');
eventSource.addEventListener('message', e => {
    console.log(e.data);
});
eventSource.addEventListener('open', e => {
    console.log('connected', e);
});
eventSource.addEventListener('error', e => {
    if (e.eventPhase === eventSource.CLOSED) {
        console.log('connection closed (..auto reconnect in 3s)');
    } else {
        console.error(e);
    }
});

// 서버에서 이벤트 이름을 지정한 경우 아래와 같이 사용 가능
/*eventSource.addEventListener('eventName', e => {
});*/

window.addEventListener('beforeunload', () => eventSource.close());
```