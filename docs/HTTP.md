# Документация: HTTP во frontend

Этот документ описывает роль HTTP в `LCMF` и обычное место HTTP runtime в
frontend-приложении.

Для общей архитектурной рамки см. [`ARCH.md`](./ARCH.md).
Для формы отдельного модуля см. [`MODULE.md`](./MODULE.md).
Для app-level слоя, который создает HTTP runtime, см. [`APP.md`](./APP.md).
Если нужен realtime transport, смотрите также [`WEBSOCKET.md`](./WEBSOCKET.md).

## 1. Назначение

HTTP в `LCMF` закрывает базовый путь связи frontend-модулей с backend.

Его роль такая:

- дать модулю устойчивый способ читать и изменять backend-состояние;
- давать единый shape ошибки на клиенте;
- переносить `correlation-id` и `request-id` в клиентскую диагностику;
- связывать результат HTTP-операции с состоянием модуля и, при необходимости,
  с публикацией сообщений в `bus`.

В `LCMF` HTTP не является внешней случайной деталью вокруг модуля. Сетевая
часть входит в ответственность модуля.

## 2. Где HTTP runtime живет в обычном приложении

HTTP runtime создается на app-level и передается `LCMF`-модулям через
зависимости.

Обычная картина такая:

- приложение создает один общий HTTP runtime;
- модули используют его для своих операций;
- модуль сам решает, как результат операции отражается в его состоянии;
- модуль при необходимости публикует сообщение в `bus`.

Небольшой пример:

Это app-level пример создания общего HTTP runtime.

```clojure
(ns app.system
  (:require [lcmf.http :as http]))

(defn make-system []
  (let [http-runtime (http/make-runtime {:base-url "/api"})]
    {:http http-runtime}))
```

Смысл этого примера простой:

- HTTP runtime принадлежит приложению;
- модули получают уже готовую инфраструктурную зависимость;
- модуль не должен каждый раз заново собирать свой собственный транспортный
  слой без необходимости.

## 3. Базовая модель HTTP-ошибки

Frontend в `LCMF` должен работать с единым приведенным видом ошибки.

Ожидаемая practically useful форма:

```clojure
{:status 409
 :code "slot_not_open"
 :message "Slot not open"
 :details nil
 :correlation-id "c1a2..."
 :request-id "r9f0..."
 :retry-after-sec nil}
```

Практические правила:

- для клиентской логики нужно ориентироваться на `code`, а не на `message`;
- `message` нужен как fallback для человека;
- `details` нужно обрабатывать как машинно-читаемую структуру;
- точные тексты ошибки нельзя считать стабильным контрактом.

## 4. `correlation-id` и `request-id`

Для HTTP в `LCMF` важно различать два идентификатора:

- `correlation-id` — идентификатор общей цепочки;
- `request-id` — идентификатор конкретного HTTP-запроса.

Практический смысл:

- `correlation-id` помогает связать frontend-ошибку с backend-трассой;
- `request-id` помогает найти конкретный запрос среди других;
- если доступны оба значения, сохранять нужно оба.

Для браузера это означает:

- JavaScript увидит эти заголовки только если приложение открыло их через
  `Access-Control-Expose-Headers`;
- practically useful минимум: `x-correlation-id, x-request-id`.

## 5. Основные функции

### `make-runtime`

Создает HTTP runtime приложения.

```clojure
(require '[lcmf.http :as http])

(def app-http
  (http/make-runtime {:base-url "/api"}))
```

Первая реализация может быть поверх:

- `fetch`;
- `XMLHttpRequest`, если это зачем-то нужно;
- thin wrapper над выбранной библиотекой клиента.

Важно:

- внешний API `LCMF` не должен зависеть от конкретного transport API;
- модуль должен работать с `request!` и приведенной ошибкой, а не с сырым
  поведением `fetch` в каждом месте отдельно.

### `request!`

Выполняет HTTP-запрос.

Сигнатура:

```clojure
(request! runtime request-map)
```

Где `request-map` обычно включает:

- `:method`
- `:path`
- `:query`
- `:headers`
- `:json` или `:body`

Пример:

Это пример использования HTTP runtime из кода `LCMF`-модуля.

```clojure
(http/request! app-http
               {:method :get
                :path "/bookings"
                :query {:user-id "u-alice"}})
```

### `normalize-error`

Приводит сырую transport-ошибку или HTTP-ответ с ошибкой к единому клиентскому
shape.

Пример:

Это пример уровня библиотеки `lcmf.http`, а не пример кода отдельного модуля.

```clojure
(defn normalize-error [response body]
  {:status (:status response)
   :code (or (:code body) "unknown_error")
   :message (or (:message body) "Request failed")
   :details (:details body)
   :correlation-id (or (get body :correlation-id)
                       (get-in response [:headers "x-correlation-id"]))
   :request-id (or (get body :request-id)
                   (get-in response [:headers "x-request-id"]))
   :retry-after-sec (parse-retry-after (:headers response))})
```

### `parse-retry-after`

Разбирает `Retry-After`, если он есть.

```clojure
(defn parse-retry-after [headers]
  (let [raw (get headers "retry-after")
        n (js/Number raw)]
    (when (and (js/Number.isFinite n) (pos? n))
      n)))
```

Это пример уровня библиотеки `lcmf.http`.

### `compute-delay-ms`

Вычисляет задержку перед следующим повтором.

```clojure
(defn compute-delay-ms [attempt retry-after-sec]
  (if retry-after-sec
    (* retry-after-sec 1000)
    (let [base (min (* 1000 (Math/pow 2 attempt)) 15000)
          jitter (rand-int 250)]
      (+ base jitter))))
```

Это пример уровня библиотеки `lcmf.http`.

## 6. Как читать основные статусы

Для `LCMF` practically useful базовая картина такая:

- `400` и `422` — ошибки входных данных, их не нужно автоматически повторять;
- `401` — сигнал потери аутентификации или сессии;
- `403` — отказ в доступе;
- `404` — отсутствие ресурса или предметный результат, а не обязательно авария;
- `409` — конфликт с текущим состоянием;
- `429` — ограничение частоты или временная блокировка;
- `503` — временная деградация или недоступность.

Важно:

- `401`, `403`, `429`, `503` являются штатными ответами;
- клиентское поведение должно ветвиться по `status` и `code`, а не по тексту;
- `429` и `503` требуют паузы перед повтором.

## 7. Повторы запросов и `Retry-After`

Если пришел `429` или `503`:

- сначала нужно проверить `Retry-After`;
- если он есть, использовать его как базовую задержку;
- если его нет, использовать fallback backoff;
- число автоматических повторов должно быть ограничено.

Пример:

Это пример вспомогательной функции уровня `LCMF`-модуля или app-level client
runtime.

```clojure
(defn retry-request! [request! attempt error]
  (let [delay-ms (compute-delay-ms attempt (:retry-after-sec error))]
    (js/setTimeout
     (fn []
       (request!))
     delay-ms)))
```

Практическое правило:

- `400`, `401`, `403`, `404` не нужно автоматически повторять как временный
  сетевой сбой;
- `429` и `503` нужно повторять только с паузой;
- повтор не должен превращаться в бесконечный цикл.

## 8. Связь HTTP и состояния модуля

HTTP-операция не завершает свою роль на уровне transport.

`LCMF`-модуль должен сам решить:

- как операция отражается в его состоянии;
- какие служебные поля состояния меняются;
- публикуется ли по итогу сообщение в `bus`.

### Пример связи HTTP и состояния

Это пример операции уровня `LCMF`-модуля.

```clojure
(defn load-bookings!
  [{:keys [state http]}]
  (swap! state assoc :load-request {:status :pending
                                    :error nil})
  (-> (http/request! http {:method :get :path "/bookings"})
      (.then (fn [bookings]
               (swap! state assoc
                      :items (into {} (map (juxt :id identity)) bookings)
                      :load-request {:status :ok
                                     :error nil})
               bookings))
      (.catch (fn [error]
                (swap! state assoc
                       :load-request {:status :error
                                      :error error})
                (throw error)))))
```

Здесь важен именно ownership: состояние обновляет сам модуль-владелец.

## 9. Связь HTTP и шины

HTTP и `bus` не конкурируют друг с другом.

Обычная картина в `LCMF` такая:

- модуль выполняет HTTP-операцию;
- модуль отражает результат в своем состоянии;
- модуль при необходимости публикует сообщение в `bus`;
- другие модули реагируют на сообщение, не зная о конкретной HTTP-операции.

### Пример связи HTTP и шины

Это пример операции уровня `LCMF`-модуля.

```clojure
(defn create-booking!
  [{:keys [state http bus]} payload]
  (-> (http/request! http
                     {:method :post
                      :path "/bookings"
                      :json payload})
      (.then (fn [booking]
               (swap! state assoc-in [:items (:id booking)] booking)
               (bus/publish! bus
                             :booking/created
                             {:id (:id booking)
                              :user-id (:user-id booking)
                              :slot-id (:slot-id booking)}
                             {:module :booking})
               booking))))
```

## 10. Что важно не делать

В `LCMF` важно не делать следующего:

- не строить клиентскую логику на точных текстах `message`;
- не разбирать HTTP-ошибки по-разному в каждом модуле;
- не терять `correlation-id` и `request-id`;
- не делать websocket заменой начального HTTP-чтения;
- не повторять `429` и `503` без паузы;
- не передавать секреты в URL и query string.

## 11. Минимальный итог

HTTP в `LCMF` — это базовый путь связи frontend-модуля с backend.

App-level слой создает общий HTTP runtime, модуль использует его для своих
операций, сам отражает результат в своем состоянии и при необходимости
продолжает цепочку через `bus`.
