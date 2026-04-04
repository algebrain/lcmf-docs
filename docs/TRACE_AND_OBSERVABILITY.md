# Документация: Трассировка и наблюдаемость

Этот документ описывает минимальный уровень трассировки и наблюдаемости,
который нужен `LCMF` для совместимости с backend, построенным по принципам
[`LCMM`](https://github.com/algebrain/lcmm-docs).

Здесь не вводится отдельная большая frontend-подсистема телеметрии. Речь идет
только о practically useful минимуме:

- сохранить `correlation-id` и `request-id`;
- не терять причинно-следственную связность внутри `bus`;
- иметь достаточно данных для bug report и support/debug payload;
- не логировать лишние секреты.

Для общей архитектурной рамки см. [`ARCH.md`](./ARCH.md).
Для HTTP-части см. [`HTTP.md`](./HTTP.md).
Для шины см. [`BUS.md`](./BUS.md).

## 1. Что здесь считается достаточным минимумом

Для `LCMF` на первом этапе достаточно следующего:

- HTTP runtime умеет читать `correlation-id` и `request-id` из ответа и ошибки;
- `bus` умеет переносить `correlation-id` и `parent-envelope` по цепочке;
- `LCMF`-модуль пишет структурированные технические сообщения через `logger`;
- при ошибке можно собрать короткий support/debug payload без секретов.

Этого достаточно, чтобы frontend можно было связать с backend-трассой и не
терять ориентиры при разборе проблем.

## 2. `correlation-id` и `request-id`

Для минимальной совместимости с
[`LCMM`](https://github.com/algebrain/lcmm-docs) frontend должен различать два
идентификатора:

- `correlation-id` — идентификатор общей цепочки;
- `request-id` — идентификатор конкретного HTTP-запроса.

Практический смысл:

- `correlation-id` помогает backend-команде найти всю цепочку;
- `request-id` помогает найти конкретный запрос;
- если доступны оба значения, сохранять нужно оба.

### Пример приведенной HTTP-ошибки

Это пример shape-а ошибки на стороне frontend runtime.

```clojure
{:status 503
 :code "dependency_unavailable"
 :message "Dependency unavailable"
 :details nil
 :correlation-id "c1a2..."
 :request-id "r9f0..."
 :retry-after-sec 5}
```

## 3. Причинно-следственная связность в `bus`

Если `LCMF`-модуль публикует сообщение в ответ на уже полученное сообщение, он
должен продолжать цепочку, а не начинать ее заново.

Практический минимум здесь такой:

- конверт шины должен содержать `correlation-id`;
- следующее сообщение должно получать `parent-envelope`;
- это позволяет сохранить связанность frontend-цепочки.

### Пример публикации следующего сообщения

Это пример внутреннего обработчика `LCMF`-модуля.

```clojure
(bus/subscribe! bus
                :booking/created
                (fn [bus envelope]
                  (bus/publish! bus
                                :notify/booking-created
                                {:booking-id (get-in envelope [:payload :id])}
                                {:parent-envelope envelope
                                 :module :notify})))
```

## 4. Что логирует `LCMF`-модуль

На первом этапе модулю достаточно писать структурированные технические
сообщения.

Практический минимум:

- `:component`
- `:event`
- важные предметные идентификаторы сценария, если они не секретны
- `:correlation-id`, если он доступен
- `:request-id`, если он доступен

### Пример технического сообщения `LCMF`-модуля

Это пример внутреннего технического сообщения модуля.

```clojure
(logger :info
        {:component ::booking
         :event :booking/create-failed
         :status 409
         :code "slot_not_open"
         :correlation-id correlation-id
         :request-id request-id})
```

Это не заменяет полноценную backend observability. Это минимальный frontend
след, достаточный для связи с backend trace.

## 5. Support/debug payload

Если пользователю или разработчику нужно передать информацию о проблеме,
frontend должен уметь собрать короткий support/debug payload.

Минимально полезный набор:

- имя сценария или экрана;
- HTTP method и endpoint, если ошибка была связана с HTTP;
- `status` и `code`;
- `correlation-id`;
- `request-id`, если он доступен;
- время ошибки.

### Пример support/debug payload

Это пример app-level payload для поддержки или bug report.

```clojure
{:screen :booking-page
 :operation :create-booking
 :method :post
 :path "/bookings"
 :status 409
 :code "slot_not_open"
 :correlation-id "c1a2..."
 :request-id "r9f0..."
 :timestamp "2026-04-04T12:00:00Z"}
```

## 6. Что нельзя включать в support/debug payload

В support/debug payload и клиентские технические сообщения нельзя включать:

- access token;
- cookie;
- session id;
- секреты из заголовков;
- сырой payload целиком, если в нем есть чувствительные данные.

Практическое правило:

- frontend должен помогать отладке;
- frontend не должен ради отладки нарушать безопасное поведение клиента.

## 7. Что здесь не нужно усложнять

На первом этапе `LCMF` не обязан:

- вводить отдельную систему метрик уровня Prometheus;
- требовать отдельный frontend observability registry;
- покрывать каждый экран и каждую функцию телеметрией;
- делать сложную span-модель внутри клиента.

Если позже появится реальная потребность в более глубокой frontend
наблюдаемости, это можно будет добавить отдельным документом или отдельным
app-level слоем. Для базовой совместимости с
[`LCMM`](https://github.com/algebrain/lcmm-docs) это не требуется.

## 8. Минимальный итог

Для `LCMF` достаточно такого минимального compliance:

- не терять `correlation-id` и `request-id`;
- продолжать причинно-следственную цепочку в `bus`;
- писать короткие структурированные технические сообщения;
- уметь собирать безопасный support/debug payload.
