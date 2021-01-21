---
layout: post
title: 基于WebFlux的WebSocket聊天功能实现
categories: [WebFlux, WebSocket]
description: 基于WebFlux的WebSocket聊天功能实现
keywords: WebFlux, WebSocket
---

# 基于WebFlux的WebSocket聊天功能实现

最近因为工作需要写了一个基于WebFlux的WebSocket聊天功能，故此记录一下思路和大致的流程。

## 1. 准备工作

代码基于JDK 14，采用SpringBoot 2.3.0.RELEASE作为基本框架，使用spring-boot-starter-webflux作为MVC框架。假设你已经基本了解以上所述框架的使用。为了方便相关依赖的管理，使用Spring Initialzr来初始话项目，基于Maven管理依赖。

本文将采取逐步深入的方式介绍基于WebFlux的WebSocket聊天功能实现，所以会略显拖沓，尽量注重入门介绍这个主的基调。

## 2. 基本思路

WebSocket是一种在单个TCP连接上进行全双工通讯的协议，也就是说能通过WebSocket实现服务器和客户端之间的即时通讯，客户端能够发送消息到服务器，服务器也能发送消息到客户端。之前要实现类似的功能，我们需要利用Ajax轮询的方式，定期向服务器获取消息，每次都会发送Http请求，浪费带宽和服务器资源。

利用WebSocket实现聊天功能，也就是基于WebSocket的全双工通讯特点，先保证每个客户和服务器之间是全双工的，再通过在发送的消息中添加发件人和收件人信息来实现客户之间消息的发送和获取。实际是客户端通过WebSocket协议将消息发送到服务器，服务器处理消息，将消息发送给目标客户端。

## 3. 实现过程

### 3.1 WebSocketHandler

WebSocketHandler是用来处理WebSocket Session的接口，在WebFlux中我们需要创建自己的Handler实现这个接口。
实现WebSocketHandler主要是要重写```Mono<Void> handle(WebSocketSession session)```方法。这个方法会在创建WebSocket连接后被调用，入参session就是对应的WebSocketSession，一般就是要处理这个session来实现特定的功能。

WebSocketHandler的逻辑是将WebSocket的全双工通讯视为一个入站流和一个出站流。WebSocketHandler的实例的```Flux<WebSocketMessage> receive()```方法能获取到这个入站流，同样的```Mono<Void> send(Publisher<WebSocketMessage> messages)```方法能够利用出站流来传递消息。WebSocketHandler将入站流和出站流整合成一个Mono，利用流本身的生命周期来指代WebSocket的生命周期，即将WebSocket的创建视为流的创建，WebSocket的关闭或异常视为流的关闭和异常，WebSocket的收发通讯视为入站流和出站流。

下面是一个简单的例子，大意是服务器接收到客户端的消息WebSocketMessage，获取其中的负载并处理，再将消息发送回客户端。

```java
@Component
public class EchoHandler implements WebSocketHandler {
    @Override
    public Mono<Void> handle(WebSocketSession session) {
        Flux<WebSocketMessage> messageFlux = session.receive().map(message -> {
            String payload = message.getPayloadAsText();
            return "Received: " + payload;
        }).map(session::textMessage);
        return session.send(messageFlux);
    }
}
```

### 3.2 注册映射路径

SpringBoot中WebFlux默认开放的是8080端口，为了实现客户端和服务器之间的连接，我们需要注册映射路径，比如在本地我们定义ws://localhost:8080/ws/echo为对应前述```EchoHandler```的路径。需要在SpringBoot中定义对应的配置类来实现这个定义。

在WebFlux中使用WebSocket只需要简单的通过在SimpleUrlHandlerMapping设置UrlMap，下面的代码就能够实现映射路径的注册。此外需要注意的是同时配置WebSocketHandlerAdapter。

```java
@Configuration
public class WebSocketConfig {
    @Bean
    public HandlerMapping webSocketMapping() {
        SimpleUrlHandlerMapping simpleUrlHandlerMapping = new SimpleUrlHandlerMapping();
        Map<String, WebSocketHandler> handlerMap = new LinkedHashMap<>();
        handlerMap.put("/ws/echo", new EchoHandler());
        simpleUrlHandlerMapping.setUrlMap(handlerMap);
        simpleUrlHandlerMapping.setOrder(-1);
        return simpleUrlHandlerMapping;
    }

    @Bean
    public WebSocketHandlerAdapter handlerAdapter() {
        return new WebSocketHandlerAdapter();
    }
}
```

到此为止就可以启动项目，连接ws://localhost:8080/ws/echo来测试能否正常使用WebSocket了。前述的EchoHandler正常情况下会接收获取到的文本数据，并将文本数据发回客户端，如发送"Hello World"，则会返回"Received: Hello World"。

### 3.3 定义消息类

一般通过WebSocket传输的数据为文本，所以我们可以采用JSON的文本来实现对消息的格式化，让消息携带更多含义，也方便前后端数据处理。WebFlux默认使用Jackson来处理JSON对应的字符串，这里为避免不必要的麻烦，就采用Jackson实现JSON转Java类。

下面的例子使用Lombok的@Data注解来简化代码长度，一共三个属性，分别为targetId、messageText、userId，对应目标的id、实际消息文本、发送人id。我们的目标是实现聊天功能，当然需要发消息和收消息的id以及消息本身。

```java
@Data
public class Message {
    private String targetId;
    private String messageText;
    private String userId;
}
```

### 3.4 消息收发

最初我们利用EchoHandler实现简单的收发功能，现在需要回到目标聊天功能上来。首先，需要通过一定方式知道当前连接的用户的身份。现在比较常用的是基于JWT来实现身份认证，但是这里为了方便实现采用最简单的方式，即直接利用query来实现。如ws://localhost:8080/ws/chat?userid=123，即告知服务器用户id为123。之后我们需要保存WebSocket连接和用户id之间的对应关系，为了实现线程安全可以使用ConcurrentHashMap来建立这个对应关系。

```java
@Component
public class ChatHandler implements WebSocketHandler {
    private static final Map<String, WebSocketSession> userMap = new ConcurrentHashMap<>();
    private static final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public Mono<Void> handle(WebSocketSession session) {
        String query = session.getHandshakeInfo().getUri().getQuery();
        Map<String, String> queryMap = getQueryMap(query);
        String userId = queryMap.getOrDefault("id", "");
        userMap.put(userId, session);
        return session.receive().flatMap(webSocketMessage -> {
            String payload = webSocketMessage.getPayloadAsText();
            Message message;
            try {
                message = objectMapper.readValue(payload, Message.class);
                String targetId = message.getTargetId();
                if (userMap.containsKey(targetId)) {
                    WebSocketSession targetSession = userMap.get(targetId);
                    if (null != targetSession) {
                        WebSocketMessage textMessage = targetSession.textMessage(message.getMessageText());
                        return targetSession.send(Mono.just(textMessage));
                    }
                }
            } catch (JsonProcessingException e) {
                e.printStackTrace();
                return session.send(Mono.just(session.textMessage(e.getMessage())));
            }
            return session.send(Mono.just(session.textMessage("目标用户不在线")));
        }).then().doFinally(signal -> userMap.remove(userId)); // 用户关闭连接后删除对应连接
    }

    private Map<String, String> getQueryMap(String queryStr) {
        Map<String, String> queryMap = new HashMap<>();
        if (!StringUtils.isEmpty(queryStr)) {
            String[] queryParam = queryStr.split("&");
            Arrays.stream(queryParam).forEach(s -> {
                String[] kv = s.split("=", 2);
                String value = kv.length == 2 ? kv[1] : "";
                queryMap.put(kv[0], value);
            });
        }
        return queryMap;
    }
}

```
