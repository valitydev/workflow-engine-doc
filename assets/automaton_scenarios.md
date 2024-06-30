# Перечень компонентов

- Automaton server (AS) - предоставляет API для доступа к control plane процессов, обеспечивает вызов процессоров 
по расписанию и в соответствии с retry policy, обеспечивает персистентность (+ отправляет оповещения в Kafka)
- Automaton client (AC) - взаимодействует с AS, инициирует старт (Start) и восстановление (Repair) процессов, осуществляет
передачу событий в процесс (Call, Notify), возвращает состояние процесса (Get), удаляет процесс (вместе с историей, Remove)
- Processor server (PS) - реализует бизнес-логику (выбор шага процесса и сами шаги), предоставляет API для доступа
к бизнес логике
- Processor client (PC) - взаимодействует с PS, осуществляет вызовы (ProcessCall, ProcessRepair) и отправку сигналов 
(ProcessSignal) в процессы

Примечание: MG реализует компоненты Automaton server и Processor client. Компоненты Processor server и Automaton client 
реализуются в сервисе-процессоре, а также могут быть реализованы раздельно

# Сценарии

## Стандартный жизненный цикл

### Старт экземпляра процесса
```mermaid
sequenceDiagram
    box Client
    participant AC
    end
    box Grey Machinegun
    participant AS
    participant PC
    end
    box Processor
    participant PS
    end

    AC->>+AS: Automaton.Start(InitArgs, [Change])
    Note over AS,PC: check existence
    Note over AS,PC: uniqueness control
    PC->>+PS: Processor.Init(InitArgs, History)
    PS-->>-PC: InitResult([Change], Action)
    Note over AS,PC: save changes
    Note over AS,PC: save action
    Note over AS,PC: schedule action
    AS-->>-AC: OK
```

### Старт экземпляра процесса - ошибка - уже существует
```mermaid
sequenceDiagram
    box Client
    participant AC
    end
    box Grey Machinegun
    participant AS
    participant PC
    end

    AC->>+AS: Automaton.Start(InitArgs, [Change])
    Note over AS,PC: check existence
    AS-->>-AC: ERROR(Already exists)
```

### Старт экземпляра процесса - ошибка процессора
```mermaid
sequenceDiagram
    box Client
    participant AC
    end
    box Grey Machinegun
    participant AS
    participant PC
    end
    box Processor
    participant PS
    end

    AC->>+AS: Automaton.Start(InitArgs, [Change])
    Note over AS,PC: check existence
    Note over AS,PC: uniqueness control
    PC-xPS: Processor.Init(InitArgs, History)
    AS-->>-AC: 5xx
```

### Срабатывание таймера
```mermaid
sequenceDiagram
    box Grey Machinegun
    participant AS
    participant PC
    end
    box Processor
    participant PS
    end

    Note over AS,PC: check action
    Note over AS,PC: read history
    PC->>+PS: Processor.ProcessSignal(TimeoutArgs, History)
    PS-->>-PC: SignalResult([Change], Action)
    Note over AS,PC: save changes
    Note over AS,PC: save action
    Note over AS,PC: schedule action
```

### Срабатывание таймера - ошибка процессора (обрабатываемая)
```mermaid
sequenceDiagram
    box Grey Machinegun
    participant AS
    participant PC
    end
    box Processor
    participant PS
    end

    Note over AS,PC: check action
    Note over AS,PC: read history
    PC-xPS: Processor.ProcessSignal(TimeoutArgs, History)
    Note over AS,PC: save retry action (with retry policy)
    Note over AS,PC: schedule retry action
```

### Срабатывание таймера - ошибка процессора (необрабатываемая)
```mermaid
sequenceDiagram
    box Grey Machinegun
    participant AS
    participant PC
    end
    box Processor
    participant PS
    end

    Note over AS,PC: check action
    Note over AS,PC: read history
    PC-xPS: Processor.ProcessSignal(TimeoutArgs, History)
    Note over AS,PC: save status - FAILED
```

### Вызов процесса
```mermaid
sequenceDiagram
    box Client
    participant AC
    end
    box Grey Machinegun
    participant AS
    participant PC
    end
    box Processor
    participant PS
    end

    AC->>+AS: Automaton.Call(CallArgs)
    Note over AS,PC: read history
    PC->>+PS: Processor.ProcessSignal(CallArgs, History)
    PS-->>-PC: SignalResult(CallResponse, [Change], Action)
    Note over AS,PC: save changes
    Note over AS,PC: save action
    Note over AS,PC: schedule action
    AS-->>-AC: CallResponse
```

### Вызов процесса - ошибка процессора (обрабатываемая)
```mermaid
sequenceDiagram
    box Client
    participant AC
    end
    box Grey Machinegun
    participant AS
    participant PC
    end
    box Processor
    participant PS
    end

    AC->>+AS: Automaton.Call(CallArgs)
    Note over AS,PC: read history
    PC-xPS: Processor.ProcessSignal(CallArgs, History)
    AS-->>-AC: 5xx
```

### Вызов процесса - ошибка процессора (необрабатываемая)
```mermaid
sequenceDiagram
    box Client
    participant AC
    end
    box Grey Machinegun
    participant AS
    participant PC
    end
    box Processor
    participant PS
    end

    AC->>+AS: Automaton.Call(CallArgs)
    Note over AS,PC: read history
    PC-xPS: Processor.ProcessSignal(CallArgs, History)
    Note over AS,PC: save status - FAILED
    AS-->>-AC: 5xx
```

### Запланировать нотификацию процесса
```mermaid
sequenceDiagram
    box Client
    participant AC
    end
    box Grey Machinegun
    participant AS
    participant PC
    end

    AC->>+AS: Automaton.Notify(NotifyArgs)
    Note over AS,PC: check existence
    Note over AS,PC: save notify action
    AS-->>-AC: OK
```

### Запланировать нотификацию процесса - ошибка получателя нотификации не существует
```mermaid
sequenceDiagram
    box Client
    participant AC
    end
    box Grey Machinegun
    participant AS
    participant PC
    end

    AC->>+AS: Automaton.Notify(NotifyArgs)
    Note over AS,PC: check existence
    AS-->>-AC: ERROR(not exist)
```

### Получение нотификации процессом
```mermaid
sequenceDiagram
    box Grey Machinegun
    participant AS
    participant PC
    end
    box Processor
    participant PS
    end

    Note over AS,PC: check notify action
    Note over AS,PC: read history
    PC->>+PS: Processor.ProcessNotify(NotifyArgs, History)
    PS-->>-PC: NotifyResult([Change], Action)
    Note over AS,PC: mark notify delivered
    Note over AS,PC: save changes
    Note over AS,PC: save action
    Note over AS,PC: scedule action
```

### Получение нотификации процессом - ошибка процессора (обрабатываемая)
```mermaid
sequenceDiagram
    box Grey Machinegun
    participant AS
    participant PC
    end
    box Processor
    participant PS
    end

    Note over AS,PC: check notify action
    Note over AS,PC: read history
    PC-xPS: Processor.ProcessNotify(NotifyArgs, History)
    Note over AS,PC: save retry action (with retry policy)
    Note over AS,PC: scedule retry action
```

### Получение нотификации процессом - ошибка процессора (необрабатываемая)
```mermaid
sequenceDiagram
    box Grey Machinegun
    participant AS
    participant PC
    end
    box Processor
    participant PS
    end

    Note over AS,PC: check notify
    Note over AS,PC: read history
    PC-xPS: Processor.ProcessNotify(NotifyArgs, History)
    Note over AS,PC: save status - FAILED
```

### Получение history процесса
```mermaid
sequenceDiagram
    box Client
    participant AC
    end
    box Grey Machinegun
    participant AS
    participant PC
    end

    AC->>+AS: Automaton.Get(ID)
    Note over AS,PC: read history
    AS-->>-AC: History
```

### Получение history процесса - ошибка - нет данных
```mermaid
sequenceDiagram
    box Client
    participant AC
    end
    box Grey Machinegun
    participant AS
    participant PC
    end

    AC->>+AS: Automaton.Get(ID)
    Note over AS,PC: read history
    AS-->>-AC: ERROR(not found)
```

### Удаление history процесса
```mermaid
sequenceDiagram
    box Client
    participant AC
    end
    box Grey Machinegun
    participant AS
    participant PC
    end

    AC->>+AS: Automaton.Remove(ID)
    Note over AS,PC: remove changes
    AS-->>-AC: OK
```

### Удаление history процесса - ошибка - нет данных
```mermaid
sequenceDiagram
    box Client
    participant AC
    end
    box Grey Machinegun
    participant AS
    participant PC
    end

    AC->>+AS: Automaton.Remove(ID)
    Note over AS,PC: remove changes
    AS-->>-AC: ERROR(not found)
```

## Сбои и восстановление процессов

### Обрабатываемый (временный) сбой в работе процесса
> **Тоже самое что и таймер = 0**

### Восстановление после обрабатываемого сбоя процессора (with retry policy)
> **Тоже самое что и таймер = следующий шаг стратегии повтора**

### Необрабатываемый сбой в работе процесса
> **Переход процесса в статус FAILED**

### Восстановление после необрабатываемого сбоя процессора (repairing)
> **Тоже самое что и вызов процесса, при успехе статус FAILED переходит в другой**

### Внутренний сбой управляющей машины
> **Неожиданная выгрузка при убивании ноды**

### Восстановление после внутреннего сбоя управляющей машины
> **Вычитка всех процессов не имеющих финального статуса по запросу в базу**