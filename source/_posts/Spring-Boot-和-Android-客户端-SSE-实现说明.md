---
title: Spring Boot 和 Android 客户端 SSE 实现说明
date: 2024-12-09 15:58:01
tags:
- Spring Boot
- SSE
- Android

categories:
- Spring Boot
- SSE
- Android
---

# Spring Boot 和 Android 客户端 SSE 实现说明

本项目实现了一个基于 Server-Sent Events (SSE) 的消息推送系统，后端使用 Spring Boot，前端使用 Android 客户端。以下是系统的整体结构和各个部分的详细说明。

## 1. Spring Boot 后端实现

后端通过 `SseEmitter` 来实现 SSE 功能。主要功能是为每个客户端维护一个 `SseEmitter` 实例，通过 `SseEmitter` 进行消息推送。

### 核心代码说明

#### 1.1 订阅接口

```kotlin
private val clients = ConcurrentHashMap<String, SseEmitter>()

fun subscribe(clientId: String): SseEmitter {
    val emitter = SseEmitter(60 * 60_000L) // 1小时超时
    clients[clientId] = emitter
    emitter.onCompletion { }
    emitter.onTimeout { clients.remove(clientId) }
    emitter.onError { clients.remove(clientId) }
    return emitter
}
```

- `clients` 是一个 `ConcurrentHashMap`，用于存储所有在线客户端的 `SseEmitter` 实例，`clientId` 作为客户端的标识。
- `subscribe` 方法会为每个客户端创建一个新的 `SseEmitter` 实例，设置超时时间为 1 小时，并且在事件超时或错误时移除该客户端。

#### 1.2 发送消息接口

```kotlin
const val RemoveClient = "RemoveClient"

fun sendEvent(clientId: String, message: String): String {
    val emitter = clients[clientId]
    return if (emitter != null) {
        try {
            when (message) {
                RemoveClient -> {
                    clients.remove(clientId)
                }

                else -> {
                    emitter.send(SseEmitter.event().name("message").data(message))
                }
            }
            "Message sent"
        } catch (e: IOException) {
            clients.remove(clientId)
            "Failed to send message"
        }
    } else {
        "Client not found"
    }
}
```

- `sendEvent` 方法用于向指定客户端发送消息。
- 根据不同的消息类型，后端会采取相应的操作，比如移除客户端或推送消息。

## 2. Android 客户端实现

Android 客户端通过 `RealEventSource` 来接收 SSE 事件，并根据消息内容执行对应的操作。

### 核心代码说明

#### 2.1 客户端连接

```kotlin
enum class SSEMessage(val func: suspend () -> Unit) {
    Logout({
        mainVM.logout()
    }),
    RemoveClient({
        VehicleApi.service.sendEvent(mainVM.user.id!!, RemoveClient.name)
    })
}

fun connect() {
    val request: Request = Request.Builder()
        .url("${baseUrl}api/v1/em/vehicleauth/subscribe/${mainVM.user.id!!}")
        .header(STORAGE_AUTHORIZATION, SpUtils.token)
        .build()

    realEventSource = RealEventSource(request, object : EventSourceListener() {

        override fun onEvent(
            eventSource: EventSource,
            id: String?,
            type: String?,
            data: String,
        ) {
            super.onEvent(eventSource, id, type, data)
            try {
                SSEMessage.valueOf(data)
            } catch (e: Exception) {
                null
            }?.apply {
                activity.lifecycleScope.launch {
                    func()
                }
            }
        }

        override fun onClosed(eventSource: EventSource) {
            super.onClosed(eventSource)
        }

        override fun onFailure(eventSource: EventSource, t: Throwable?, response: Response?) 		 {
            super.onFailure(eventSource, t, response)
        }

        override fun onOpen(eventSource: EventSource, response: Response) {
            super.onOpen(eventSource, response)
        }
    })
    realEventSource.connect(client)
}
```

- `connect` 方法用于建立与服务器的 SSE 连接。
- `RealEventSource` 是 `OkHttp` 的一个扩展类，能够处理 SSE 协议。
- 在接收到事件时，客户端会根据事件的 `data` 字段来解析消息类型（如 `Logout`、`RemoveClient`），并执行相应的操作。

#### 2.2 事件处理

```kotlin
enum class SSEMessage(val func: suspend () -> Unit) {
    Logout({
        mainVM.logout()
    }),
    RemoveClient({
        VehicleApi.service.sendEvent(mainVM.user.id!!, RemoveClient.name)
    })
}
```

- `SSEMessage` 枚举类定义了两种消息类型：
    - `Logout`: 触发用户登出操作。
    - `RemoveClient`: 通知服务器移除客户端。
- 每个消息类型关联一个挂起函数 (`func`)，该函数会在接收到事件时执行。

## 3. 使用说明

### 3.1 后端配置

在 Spring Boot 后端中，通过 `SseEmitter` 实现了 SSE 功能。前端只需要调用 `subscribe` 方法来订阅特定客户端，然后使用 `sendEvent` 方法发送消息。

### 3.2 客户端配置

在 Android 客户端中，通过 `RealEventSource` 类接收来自后端的 SSE 消息，并根据消息类型执行相应的操作（如登出或移除客户端）。客户端连接会在 `connect` 方法中建立，消息接收后会根据 `SSEMessage` 枚举类型来触发不同的操作。

## 4. 总结

本实现通过 SSE 协议实现了实时消息推送，结合了 Spring Boot 后端和 Android 客户端。后端使用 `SseEmitter` 来处理与多个客户端的连接，而客户端使用 `RealEventSource` 接收并处理事件。通过此方式，前后端实现了实时通信，确保系统能够及时响应客户端的请求。

### 关键点

- **后端使用 `SseEmitter`** 实现 SSE 消息推送。
- **客户端使用 `RealEventSource`** 接收实时事件并处理。
- **消息类型通过枚举 `SSEMessage` 定义**，并结合挂起函数执行不同的操作。

这样就实现了一个高效、实时的消息推送系统，适用于需要实时通知和更新的场景。