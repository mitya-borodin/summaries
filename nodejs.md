## Не блокирующий I/O в NodeJS

Подробное видео объяснение <https://www.youtube.com/watch?v=zDlg64fsQow&list=PL6DxKON1uLOHyPVa0XhcqcW2RhDytDpAI&index=10>

**Основне компоненты:**
- [V8](https://medium.com/nuances-of-programming/движок-javascript-что-внутри-f0db9b988b90)
- [LibUV](https://imnotgenius.com/21-sobytijnyj-tsikl-biblioteka-libuv/?ysclid=lgxbin3g40962538536)

**V8**
- Транслирует JS код в машинный код
- Выделяет память для объектов
- Занимается сборкой мусора

**V8** говорить nodejs, чтобы он что-то сделал в ОС, с файлом, с сетью, а nodejs передает запрос в **libUV**.
И тем самым достигается кросс платформеность, так как **libUV** знает как работать на windows, linux, macos.

**LibUV**
- Кроссплатформенный I/O
- - Файлы
- - Сеть
- Event loop

Под капотом использует 4 потока. 
Ну или как говорит ChatGPT: "количество потоков в пуле рабочих потоков libuv определяется при запуске и по умолчанию равно количеству ядер процессора."

## **Многопоточный ввод/вывод**

Показать пример необлокирующего ввода вывода, можо через встроенную библиотеку `crypto`, запуская 5 вызово в цикле
и смотреть на сколько быстро вызываются колбэки. 
Пятый колбэк всегда будет выполняться позже первых четырех, так как на него нехватает потоков в LibUV.

Таким образом поучается, что nodejs обнопоточный, а libUV позволяет работать в многопоточном режиме, для взаимодейтсвия с сетью, диском и другими возможностями ОС.

## **Устройство не блокирующего I/O**

Неблокирующий ввод выод основан на шаблоне "Reactor" - это архитектурный шаблон проектирования, используемый для асинхронной обработки сервисных запросов, которые поступают от различных клиентов к одному или нескольким обслуживающим модулям. Это особенно полезно в ситуациях, где у вас есть множество открытых соединений, которые в основном ожидают и должны быстро реагировать на события, когда они происходят.

Шаблон "Reactor" обычно состоит из следующих компонентов:

- **Reactor:** Этот компонент управляет регистрацией и удалением обработчиков и их событий, и он также диспетчеризует события обработчикам. Это, по сути, ядро шаблона Reactor.
- **Обработчики (Handlers):** Обработчики - это объекты, которые реагируют на определенные типы событий. Каждый обработчик имеет определенную реализацию, которая определяет, как он будет реагировать на свое событие.
- **События (Events):** События - это определенные условия, которые обработчики могут обрабатывать. Когда событие происходит, Reactor диспетчеризует его соответствующему обработчику.
- **Демультиплексор событий (Event Demultiplexer):** Это компонент, который ожидает наступления любого из зарегистрированных событий. Как только одно из этих событий происходит, демультиплексор событий уведомляет Reactor, который затем диспетчеризует событие соответствующему обработчику.Он эффективен, потому что позволяет приложению ожидать данные от множества источников ввода-вывода без необходимости постоянно опрашивать каждый источник на предмет наличия данных. Вместо этого демультиплексор событий оповещает приложение, когда данные доступны, что позволяет приложению использовать свои ресурсы более эффективно.Различные операционные системы предоставляют разные механизмы для демультиплексации событий. Например, в Unix-подобных системах часто используются механизмы select, poll и epoll, а в Windows - I/O Completion Ports.

Библиотека libuv, которая используется в Node.js, и технология Twisted в Python - это примеры реализаций шаблона Reactor. Они используют шаблон Reactor для предоставления эффективного механизма асинхронного ввода-вывода.

**Этапы работы**
1. Приложение создает операцию I/O, и поределяет для неё обработчик передав запрос демультиплексору событий.
2. После совержения операции I/O, мультиплексор добавлет эти события в `очередь событий`.
3. Event loop обходит очередь событий.
4. Для каждого события вызывается обработчик и попадает в стек вызовов. Но если в обработчике будет долгое вычисление, то EL заблокируется вместе с основным потоком, и не сможет вызвать следующий обработчик.
5. В обработчике события который был вызыван EL, может быть оконечная обработка и поток освобождается и управление переходит к EL, или в обработчике может быть ещё один вызов I/O и событие добавляется в демультиплексор событий.

## **Event loop**
Плюсы:
- Дает большое удобноство, так как нам не нужно управлять потоками.
- Дает большую скорость работы над простыми операциями ввода/вывода. Такую скорость нельзя получить в системах с блокирующим I/O, и в многопоточных системах, так как издержки на управления потоками будут выше чем в случае с Event loop.

Минусы:
- Много асинхронного кода.
- Любое сложное вычисление, занимает поток и приложение блокируется.

Составные части:
- Приложение - Находится в V8
- Демультиплексор событий - Находится в ОС
- Call stack - находится в V8
- Event queue (Macro Tasks) - в эту очередь попадает все, кроме промисов. Находится в LibUV.
- Task queue (Micro tasks) - в эту очередь попадают только промисы. Имеет приоритет на выполнение перед Event queue. Находится в LibUV.

За очереди отвечает event loop, а за call stack ответчает V8.

Как только call stack освобождается, event loop берет задачи из Task queue (Micro tasks), а после чего из Event queue (Macro Tasks). В случае если в макро задаче, будет создана микро задача, то после выполнения макро задачи будет взята микро задача.

Фазы:
- Таймеры - в этой фазе выполняются call back которые запланирвоаны setTimeout, setInterval.
- I/O call back - выполняются все call back, за исключением таймеров, колбэков `close`, и события которые были определены `setImediate`.
- Ожидание, подготовка - используется только для внутренних целей.
- Опрос - в ней происходит получение новых событий I/O при этом nodejs может блокироваться на этом этапе.
- Проверка - тут выполняются все события которые поеределны через `setImediate`
- Колбэки `close` - например закрытие web socket соединения, закрытие стрима и другие события `close`.

## **Worker threads / Node cluester / Spawned node process**

- **Worker Threads:** В Node.js 10.5.0 была добавлена поддержка worker threads как экспериментальной функции, а с версии 12 она стала стабильной. Worker threads позволяют выполнять CPU-интенсивные операции без блокировки основного потока. Это означает, что вы можете выполнять задачи, которые требуют много процессорного времени, в отдельных потоках, не замедляя остальную часть вашего приложения.
- **Node Clusters:** Кластеры в Node.js - это способ создания нескольких экземпляров вашего приложения, которые могут работать параллельно. Это особенно полезно, если у вас есть многоядерный процессор, так как каждый экземпляр приложения будет работать на своем ядре. Модуль cluster Node.js позволяет легко создавать эти кластеры.
- **Spawned Node Processes:** Метод spawn модуля child_process в Node.js позволяет создать новый процесс с новым экземпляром Node.js (или любой другой командой). Это похоже на кластеры, но предоставляет гораздо больше контроля и больше изоляции между процессами. Однако это также требует больше ресурсов, так как каждый процесс должен иметь свое собственное окружение выполнения.

Все три подхода могут быть полезны в разных ситуациях. Worker threads являются полезными для CPU-интенсивных задач, которые не блокируют основной поток. Кластеры полезны, когда у вас есть многоядерный процессор и вы хотите максимально его использовать. Spawned processes полезны, когда вам нужна полная изоляция между различными частями вашего приложения.

## Streams

**Основные типы**
- Readable - Чтение
- Wrateable - Запись
- Duplex - Чтение + Запись, Readable + Wrateable
- Transform - Такой же как Duplex, но может изменить данные

Предназначены, чтобы передавать информацию малыми кусочками.
По умолчанию один такой кусочек весит 64 KB.

Для того, чтобы синхронизировать скорость потока на чтение с диска, и потока записи в сеть, используется метод `pipe`, он не будет читать новый кусок, пока текущий не будет записан.
