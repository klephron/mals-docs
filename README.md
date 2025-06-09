# docs

## Architecture

В данном разделе описана архитектура системы в целом и для компонентов по отдельности, а также представленны диаграммы.

### System

![](./diagrams/architecture.drawio.png)

Разработанная система состоит из следующих компонентов:

- `mals-vscode` - расширение для `VSCode`, `VSCodium` и прочих сборок (далее `VSCode`), настраивающее и запускающее адаптер `LSP` `mals-lsp-adapter` и реализующее специфичные для `VSCode` функции.

- `mals-lsp-adapter` - клиент, обеспечивающий базовую функциональность системы, коммуницирующий с тектовыми редакторами по `stdio` согласно спецификации `LSP`. Запускается непосредственно текстовым редактором.

- `mals-engine` - сервис, реализующий основную логику системы. Общается с `LSP` серверами, таким образом получая семантическую информацию о проекте, и с `LLM` серверами, дополнительно расширяющими ответы.
    - Запускает `LSP` сервера по необходимости клиентов.
    - Подключается к `LLM` серверам, общаясь согласно стандартам (например, `OpenAI API spec`).

Таким образом, через `mals-lsp-adapter` реализуется функциональность, которая будет сразу поддерживаться текстовыми редакторами, которые поддерживают `LSP`, а для расширения этой функциональности для конкретных текстовых редакторов реализуются плагины или расширения (например, как для `VSCode` расширение `mals-vscode`).

### `mals-lsp-adapter`

![](./diagrams/lsp-adapter-activity.drawio.png)

На диаграмме `LSP Message passing from mals-vscode to mals-engine` описывается один из основных прецедентов компонента. На нем показан процесс обработки запросов от пользователей:

1. Поступает `LSP` сообщение от пользователя.
2. Это сообщение парсится (для считывания, поскольку с клиентом общение происходит через `stdio`) и валидируется.
3. Из запроса пользователя формируются сообщения для отправки на `mals-engine`, так как `mals-engine` не гарантирует общение по спецификации `LSP`.
4. Сформированные сообщения отправляются на `mals-engine`.

Обратный процесс отправки сообщений аналогичен рассмотренному выше, где одно и более сообщений от `mals-engine` конвертируются в формат, определяемый `LSP`, и отправляются согласно той же спецификации.

Благодаря этому осуществляется декомпозиция системы, что упрощает разработку `mals-engine`.

### `mals-engine`

![](./diagrams/engine-completion.drawio.png)

На диаграмме `Completion Request Processing` описывается прецедент автодополнения.

1. Клиент отправляет запрос на автодополнение.
2. Осуществляется поиск запроса по `Workspace`, так как каждый пользователь может иметь несколько активных рабочих проектов.
3. Выполняется поиск `LSP` серверов, результаты которых могут понадобиться для генерации ответа (например, если файл `python`, то может быть найден сервер `pyright`, если он был сконфигурирован).
4. Каждому `LSP` серверу отправляется `LSP` запрос на автодополнение.
5. После того, как были получены все ответы от `LSP` серверов, из их ответов и контекстной информации о проекте составляется промпт для модели.
6. Выбранной модели отправляется промпт и ожидается ответ.
7. Ответ модели валидируется и результат автодополнения отправляется клиенту.

## Features

В данном разделе описываются основные функциональности реализованных компонентов.

### `mals-vscode`

- Конфигурация `mals-lsp-adapter` и его запуск через `settings.json` и настройки `VSCode`.

> Платформоспецифичные функции не реализованы.

### `mals-lsp-adapter`

- Общение с клиентами (текстовыми редакторами) через `stdio`
- Общение с `mals-engine` через `TCP, bidirectional`.

Прочее:

- Логирование (файл)
- Параметризация запуска, смотреть описание через передачу параметра `-h`.

> Текущая функциональность адаптера - `proxy` без модификации запросов/ответов.

### `mals-engine`

- Независимая обработка запросов от нескольких клиентов
- Поддержка нескольких `workspace` для каждого клиента
- Настройка конфигурации (`LLM` серверов, `LSP` серверов и `workspace` клиентов) через конфигурационный файл
- Генерация автодополнения с использованием выбранной `LLM` (смотреть `internal/workspace/completion.go` и `internal/workspace/completion_prompt.go`)

LSP:

- Полная синхронизация файлов от клиентов `TextDocumentSync FULL`.
- Автодополнение `CompletionProvider`

Прочее:

- Логирование (stdout)
- Параметризация запуска, смотреть описание через передачу параметра `-h`.


## Tests

В данном разделе представлены отчеты о тестировании и их описание.

### `mals-vscode`

![](./tests/mals-vscode-test-report.png)

- `Extension Tests` - тесты возможностей запущенного расширения
- `Configuration Tests` - тесты конфигурации расширения
- `Activation Tests` - модульные тесты процесса активации расширения

### `mals-lsp-adapter`

```
--- PASS: TestScannerSplit (0.00s)
    --- PASS: TestScannerSplit/complete_valid_message (0.00s)
    --- PASS: TestScannerSplit/incomplete_message_no_separator (0.00s)
    --- PASS: TestScannerSplit/incomplete_message_partial_content (0.00s)
    --- PASS: TestScannerSplit/zero_length_content (0.00s)
    --- PASS: TestScannerSplit/invalid_content_length (0.00s)
    --- PASS: TestScannerSplit/exact_content_length_match (0.00s)
    --- PASS: TestScannerSplit/content_longer_than_specified (0.00s)
    --- PASS: TestScannerSplit/multiple_messages_in_buffer (0.00s)
    --- PASS: TestScannerSplit/empty_input (0.00s)
PASS
ok      mals-lsp-adapter/test   0.002s
```

### `mals-engine`

```
--- PASS: TestScannerSplit (0.00s)
    --- PASS: TestScannerSplit/complete_valid_message (0.00s)
    --- PASS: TestScannerSplit/incomplete_message_no_separator (0.00s)
    --- PASS: TestScannerSplit/incomplete_message_partial_content (0.00s)
    --- PASS: TestScannerSplit/zero_length_content (0.00s)
    --- PASS: TestScannerSplit/invalid_content_length (0.00s)
    --- PASS: TestScannerSplit/exact_content_length_match (0.00s)
    --- PASS: TestScannerSplit/content_longer_than_specified (0.00s)
    --- PASS: TestScannerSplit/multiple_messages_in_buffer (0.00s)
    --- PASS: TestScannerSplit/empty_input (0.00s)

--- PASS: TestDecodeRequest (0.00s)
    --- PASS: TestDecodeRequest/valid_request (0.00s)
    --- PASS: TestDecodeRequest/request_with_different_method (0.00s)
    --- PASS: TestDecodeRequest/no_separator (0.00s)
    --- PASS: TestDecodeRequest/invalid_content_length (0.00s)
    --- PASS: TestDecodeRequest/invalid_JSON (0.00s)
    --- PASS: TestDecodeRequest/negative_ID (0.00s)
    --- PASS: TestDecodeRequest/zero_ID (0.00s)

--- PASS: TestDecodeNotification (0.00s)
    --- PASS: TestDecodeNotification/valid_notification (0.00s)
    --- PASS: TestDecodeNotification/notification_with_different_method (0.00s)
    --- PASS: TestDecodeNotification/no_separator (0.00s)
    --- PASS: TestDecodeNotification/invalid_JSON (0.00s)
    --- PASS: TestDecodeNotification/invalid_content_length (0.00s)
    --- PASS: TestDecodeNotification/long_method_name (0.00s)

    --- PASS: TestEncode/encode_request (0.00s)
    --- PASS: TestEncode/encode_notification (0.00s)
    --- PASS: TestEncode/encode_response (0.00s)
    --- PASS: TestEncode/encode_request_with_negative_ID (0.00s)
    --- PASS: TestEncode/encode_LSP_method (0.00s)
    --- PASS: TestEncode/unencodable_input (0.00s)

--- PASS: TestUrlFile (0.00s)
--- PASS: TestUrlDirectory (0.00s)
--- PASS: TestUrlDirectorySlash (0.00s)

--- PASS: TestGetLastNChars (0.00s)
    --- PASS: TestGetLastNChars/Simple_single_line_-_get_3_chars (0.00s)
    --- PASS: TestGetLastNChars/Single_line_-_get_more_chars_than_available (0.00s)
    --- PASS: TestGetLastNChars/Multi-line_-_position_at_start_of_second_line (0.00s)
    --- PASS: TestGetLastNChars/Multi-line_-_position_in_middle_of_second_line (0.00s)
    --- PASS: TestGetLastNChars/Multi-line_-_get_last_chars_crossing_lines (0.00s)
    --- PASS: TestGetLastNChars/Empty_content (0.00s)
    --- PASS: TestGetLastNChars/Position_at_start_of_content (0.00s)
    --- PASS: TestGetLastNChars/Line_beyond_content_length (0.00s)
    --- PASS: TestGetLastNChars/Char_beyond_line_length (0.00s)
    --- PASS: TestGetLastNChars/Zero_characters_requested (0.00s)
    --- PASS: TestGetLastNChars/Multi-line_with_empty_lines (0.00s)
```
