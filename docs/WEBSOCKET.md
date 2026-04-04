# Документация: WebSocket во frontend

Этот документ описывает роль websocket в `LCMF`.

Для общей архитектурной рамки см. [`ARCH.md`](./ARCH.md).
Для app-level слоя, который владеет transport-инфраструктурой, см.
[`APP.md`](./APP.md).
Для шины, через которую модули координируются между собой, см.
[`BUS.md`](./BUS.md).

## 1. Главный принцип

В `LCMF` websocket является transport уровня приложения.

Это означает:

- websocket endpoint принадлежит app-level слою;
- `LCMF`-модуль не владеет websocket transport;
- `LCMF`-модуль публикует и потребляет сообщения через `bus`;
- приложение принимает внешние websocket-сообщения и доставляет push в
  браузер.

Именно в таком порядке это и нужно понимать. WebSocket не становится прямым
API отдельного модуля.

## 2. Когда websocket нужен

WebSocket нужен тогда, когда приложению действительно нужен push в браузер.

Базовый practically useful сценарий такой:

- начальное состояние загружается по HTTP;
- websocket доставляет последующие обновления;
- после reconnect клиент при необходимости перечитывает состояние по HTTP.

Это означает:

- websocket не заменяет HTTP;
- экран не должен зависеть только от постоянного сокет-соединения;
- потеря соединения не должна делать приложение логически неработоспособным.

## 3. Что принадлежит приложению

App-level слой владеет:

- websocket endpoint;
- handshake policy;
- `Origin` policy;
- auth и transport-level authorization;
- хранением соединений и подписок;
- browser-facing формой сообщений;
- topic routing;
- доставкой сообщений подписчикам;
- app-owned ingress event для входящих сообщений.

### Пример app-level создания websocket runtime

Это app-level пример.

```clojure
(defn make-system []
  (let [ws-runtime (ws/make-runtime)]
    {:ws ws-runtime}))
```

## 4. Чем владеет `LCMF`-модуль

`LCMF`-модуль владеет:

- своими предметными сообщениями;
- своей логикой состояния;
- своей доменной интерпретацией входящих фактов;
- своими HTTP contract-ами;
- своими bus contract-ами;
- своими read contract-ами.

`LCMF`-модуль не должен владеть:

- websocket endpoint;
- browser-facing topic model;
- browser-facing shape сообщения;
- raw websocket input contract.

## 5. Исходящий поток в браузер

Каноническая модель исходящего потока такая:

- `LCMF`-модуль публикует предметное сообщение в `bus`;
- app-level websocket слой подписывается на это сообщение;
- приложение решает, нужно ли доставлять его в браузер;
- приложение строит browser-facing сообщение;
- приложение отправляет его через websocket transport.

### Пример сообщения, которое публикует `LCMF`-модуль

Это пример уровня `LCMF`-модуля.

```clojure
(bus/publish! bus
              :booking/created
              {:id booking-id
               :slot-id slot-id
               :user-id user-id}
              {:module :booking})
```

### Пример app-level проекции сообщения в browser-facing websocket message

Это app-level пример.

```clojure
(bus/subscribe! bus
                :booking/created
                (fn [_ envelope]
                  (let [user-id (get-in envelope [:payload :user-id])
                        message {:type "event"
                                 :event "booking/created"
                                 :topic ["user" user-id]
                                 :payload {:bookingId (get-in envelope [:payload :id])
                                           :userId user-id
                                           :slotId (get-in envelope [:payload :slot-id])}
                                 :correlationId (:correlation-id envelope)}]
                    (ws/send-to-topic! ws-runtime
                                       [:user user-id]
                                       message))))
```

Во втором примере именно приложение решает, что из предметного сообщения уйдет
в браузер и в каком виде.

## 6. Входящий поток из браузера

Если websocket используется не только для push, но и для входящих сообщений,
приложение не должно публиковать доменные сообщения от имени модуля.

Каноническая модель входящего потока такая:

- браузер присылает websocket message;
- приложение проверяет transport-level правила;
- приложение нормализует message;
- приложение публикует app-owned ingress event в `bus`;
- `LCMF`-модуль при необходимости подписывается на это ingress event и сам
  решает, какова его предметная интерпретация.

Зафиксированное ingress event:

- `:app/external-message-received`

### Пример app-owned ingress payload

Это пример app-level contract-а, который может читать `LCMF`-модуль.

```clojure
{:message-kind :command
 :message-name "booking/create"
 :payload {:slot-id "slot-09-00"}
 :subject {:user-id "u-alice"}
 :source {:channel :websocket
          :transport :ws
          :session-id "..."
          :remote-addr "..."
          :origin "http://localhost:3006"}
 :correlation-id "..."
 :received-at 1710000000}
```

Важно:

- это app-owned contract;
- это не raw websocket frame;
- это не доменное сообщение `LCMF`-модуля.

### Пример реакции `LCMF`-модуля на ingress event

Это пример уровня `LCMF`-модуля.

```clojure
(bus/subscribe! bus
                :app/external-message-received
                (fn [_ envelope]
                  (let [{:keys [message-name payload]} (:payload envelope)]
                    (when (= message-name "booking/create")
                      (create-booking! {:slot-id (:slot-id payload)}))))
                {:module :booking})
```

## 7. Минимальный scope первой фазы

Первая фаза websocket в `LCMF` включает:

- `subscribe`;
- `unsubscribe`;
- `ping`;
- server push от приложения к браузеру.

Первая фаза не обязана включать:

- произвольный command API по websocket как основной путь системы;
- durable delivery;
- offline queue;
- распределенное хранение соединений.

## 8. Browser-facing contract

Внутреннее предметное сообщение и browser-facing websocket message не обязаны
совпадать один к одному.

Приложение может:

- убрать лишние поля;
- переименовать поля в удобную внешнюю форму;
- добавить технические поля вроде `correlationId`.

Приложение не должно:

- менять предметный смысл события;
- придумывать новый предметный статус, которого модуль не объявлял.

### Пример browser-facing сообщений первой фазы

Это пример app-level websocket contract-а.

```json
{"type":"subscribed","topic":["user","u-alice"]}
{"type":"unsubscribed","topic":["user","u-alice"]}
{"type":"pong"}
{"type":"error","code":"subscription_rejected","message":"Subscription rejected"}
{"type":"event","event":"booking/created","topic":["user","u-alice"],"payload":{"bookingId":"...","userId":"u-alice","slotId":"..."},"correlationId":"..."}
```

## 9. Reconnect и resync

Reconnect сам по себе не гарантирует восстановление точного состояния.

Практическая модель в `LCMF` такая:

- приложение или экран восстанавливает соединение;
- подписки создаются заново;
- если нужна точная картина, состояние перечитывается по HTTP;
- только после этого экран снова считается согласованным.

### Пример app-level поведения после закрытия соединения

Это app-level пример.

```clojure
(defn on-close! [{:keys [http reconnect! reload-screen-state!]}]
  (reload-screen-state! http)
  (reconnect!))
```

## 10. Безопасность

Минимальный baseline такой:

- auth на handshake уровне приложения;
- проверка `Origin`;
- отсутствие секретов в URL и query string;
- authorization на каждый `subscribe`;
- единый безопасный отказ без лишних деталей;
- reconnect проходит ту же проверку заново;
- raw payload не логируется по умолчанию.

## 11. Что важно не делать

В `LCMF` важно не делать следующего:

- не давать `LCMF`-модулю прямую зависимость от websocket transport;
- не публиковать доменные сообщения модуля от имени `app`;
- не превращать raw websocket payload в контракт модуля;
- не подменять HTTP начальной загрузки одним websocket;
- не смешивать transport decisions и доменную интерпретацию.

## 12. Минимальный итог

WebSocket в `LCMF` — это app-level transport, который дополняет HTTP.

`LCMF`-модуль не владеет websocket endpoint и не должен знать transport-детали.
Он публикует и потребляет сообщения через `bus`, а приложение само решает, как
доставлять их в браузер и как принимать входящие websocket-сообщения.
