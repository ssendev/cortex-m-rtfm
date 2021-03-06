# Очередь таймера

Когда включена опция `timer-queue`, фреймворк RTFM включает
*глобальную очередь таймера*, которую приложения могут использовать, чтобы
*планировать* программные задачи на запуск через некоторое время в будущем.

Чтобы была возможность планировать программную задачу, имя задачи должно
присутствовать в аргументе `schedule` контекста атрибута. Когда задача
планируется, момент ([`Instant`]), в который задачу нужно запустить, нужно передать
как первый аргумент вызова `schedule`.

[`Instant`]: ../../api/rtfm/struct.Instant.html

Рантайм RTFM включает монотонный, растущий только вверх, 32-битный таймер,
значение которого можно запросить конструктором `Instant::now`. Время ([`Duration`])
можно передать в `Instant::now()`, чтобы получить `Instant` в будущем. Монотонный
таймер отключен пока запущен `init`, поэтому `Instant::now()` всегда возвращает
значение `Instant(0 /* циклов тактовой частоты */)`; таймер включается сразу перед
включением прерываний и запуском `idle`.

[`Duration`]: ../../api/rtfm/struct.Duration.html

В примере ниже две задачи планируются из `init`: `foo` и `bar`. `foo` -
запланирована на запуск через 8 миллионов тактов в будущем. Кроме того, `bar`
запланирован на запуск через 4 миллиона тактов в будущем. `bar` запустится раньше
`foo`, т.к. он запланирован на запуск первым.

> **ВАЖНО**: Примеры, использующие API `schedule` или абстракцию `Instant`
> **не** будут правильно работать на QEMU, потому что функциональность счетчика
> тактов Cortex-M не реализована в `qemu-system-arm`.

``` rust
{{#include ../../../../examples/schedule.rs}}
```

Запуск программы на реальном оборудовании производит следующий вывод в консоли:

``` text
{{#include ../../../../ci/expected/schedule.run}}
```

## Периодические задачи

Программные задачи имеют доступ к `Instant` в момент, когда были запланированы
на запуск через переменную `scheduled`. Эта информация и API `schedule` могут
быть использованы для реализации периодических задач, как показано в примере ниже.

``` rust
{{#include ../../../../examples/periodic.rs}}
```

Это вывод, произведенный примером. Заметьте, что есть смещение / колебание нуля
даже если `schedule.foo` была вызвана в *конце* `foo`. Использование
`Instant::now` вместо `scheduled` имело бы влияние на смещение / колебание.

``` text
{{#include ../../../../ci/expected/periodic.run}}
```

## Базовое время

Для задач, планируемых из `init` мы имеем точную информацию о их планируемом
(`scheduled`) времени. Для аппаратных задач нет `scheduled` времени, потому
что эти задачи асинхронны по природе. Для аппаратных задач рантайм предоставляет
время старта (`start`), которе отражает время, в которое обработчик прерывания
был запущен.

Заметьте, что `start` **не** равен времени возникновения события, вызвавшего
задачу. В зависимости от приоритета задачи и загрузки системы время
`start` может быть сильно отдалено от времени возникновения события.

Какое по Вашему мнению будет значение `scheduled` для программных задач которые
*вызываются*, вместо того чтобы планироваться? Ответ в том, что вызываемые
задачи наследуют *базовое* время контекста, в котором вызваны. Бызовым для
аппаратных задач является `start`, базовым для программных задач - `scheduled`
и базовым для `init` - `start = Instant(0)`. `idle` на сомом деле не имеет
базового времени но задачи, вызванные из него будут использовать `Instant::now()`
как их базовое время.

Пример ниже демонстрирует разное значение *базового времени*.

``` rust
{{#include ../../../../examples/baseline.rs}}
```

Запуск программы на реальном оборудовании произведет следующий вывод в консоли:

``` text
{{#include ../../../../ci/expected/baseline.run}}
```
