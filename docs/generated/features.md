# Возможности 1C Database MCP

Карта создана из типизированных категорий настоящего реестра `tools/list`. Она описывает функции сервиса, а не стадии выполнения команд.

```mermaid
flowchart LR
    service["1C Database MCP"]
    area0["Проекты и состояние"]
    service --> area0
    area1["Подключения"]
    service --> area1
    area2["Конфигурация и расширения"]
    service --> area2
    area3["Синхронизация"]
    service --> area3
    area4["Хранилище 1С"]
    service --> area4
    area5["XML объекты"]
    service --> area5
    area6["CF и CFE"]
    service --> area6
    area7["ibcmd и расширения"]
    service --> area7
    area8["RAC и сеансы"]
    service --> area8
    area9["Операции и логи"]
    service --> area9
```

- [Все опубликованные MCP инструменты по группам](tool-catalog.md)
- [Как устроены подключённые БД и где взять фактическую карту](database-topology.md)
- [Технические схемы исполнения](load-workflows.md)
