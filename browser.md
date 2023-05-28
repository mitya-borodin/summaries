# Не блокирующий I/O браузере

Подробное объяснение <https://www.youtube.com/watch?v=zDlg64fsQow&list=PL6DxKON1uLOHyPVa0XhcqcW2RhDytDpAI&index=10> 

**Движок решает следующие задачи:**
- Heap и Call stack
- Работа с памятью и GC
- Компиляция JS в машинный код

**Браузер решает следующие задачи:**
- Event loop
- Event queue (Macro Tasks)
- Task queue (Micro tasks)
- Web API

**Составные части не блокирующего I/O**
- Event loop - не является частю движка JS, предоставлется средой, браузером или nodejs. Устройство в браузере и nodejs разное.
- Call stack - первый зашел последний вышел
- Event queue (Macro Tasks) - в эту очередь попадает все, кроме промисов.
- Task queue (Micro tasks) - в эту очередь попадают только промисы. Имеет приоритет на выполнение перед Event queue.
- Web API - предоставляет timeout, interval, обработку событий ввода, сети. 

За очереди отвечает event loop, а за call stack ответчает V8.

Как только call stack освобождается, event loop берет задачи из Task queue (Micro tasks), а после чего из Event queue (Macro Tasks).
В случае если в макро задаче, будет создана микро задача, то после выполнения макро задачи будет взята микро задача.

**Пораждать Task queue (Micro tasks) могут:**
- Промисы
- QueueMicrotasks - Явное создание микро таски.
- MutationObserver - Инструмент который позволяет следить за DOM nodes

**Пораждать Event queue (Macro Tasks) могут:**
- Таймеры
- События ввода, загрузки и так далее.
- Барузером, события рендара, сеть и прочее. 

# Стадии рендера

- Style calculation - применение селекторов к элементам.
- 
