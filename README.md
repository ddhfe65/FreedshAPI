# FreedshAPI

<p align="center">
  <strong>Agentic OpenAI-Compatible API Proxy for DeepSeek Web Chat</strong><br>
  <em>Локальный API-прокси для DeepSeek Web, оптимизированный для автономных кодинг-агентов (dsh, Claude Code, Open WebUI)</em>
</p>

<p align="center">
  <img alt="Node.js 18+" src="https://img.shields.io/badge/node-18%2B-339933.svg" />
  <img alt="Zero npm dependencies" src="https://img.shields.io/badge/dependencies-0-blue.svg" />
  <img alt="Tests Passing" src="https://img.shields.io/badge/tests-53%20passed-brightgreen.svg" />
  <img alt="OpenAI Compatible" src="https://img.shields.io/badge/OpenAI-compatible-black.svg" />
  <img alt="License MIT" src="https://img.shields.io/badge/license-MIT-green.svg" />
</p>

<p align="center">
  <a href="#english">English Documentation</a> • <a href="#русский">Документация на русском</a>
</p>

---

<a name="english"></a>
## English

### Overview

**FreedshAPI** is a high-performance, lightweight local API gateway wrapping **DeepSeek Web Chat** (`chat.deepseek.com`). It allows you to use your standard DeepSeek Web account (via local browser session) as a full drop-in replacement for OpenAI `/v1/chat/completions`, Anthropic `/v1/messages`, and OpenAI Responses endpoints.

While standard web-chat proxies work well for simple one-off questions, they quickly break down when used with autonomous coding agents (like **dsh / DeepSeek Harness**, **Claude Code**, or **Aider**). **FreedshAPI** solves these problems with custom-engineered multi-turn session tree synchronization, resilient tool-call parsing, and streaming repair.

### Why FreedshAPI? (Core Engineering Fixes)

1. **Multi-Turn Delta Synchronization (No Context Bloat)**
   * *Problem:* Agentic clients send full conversation history on every turn. Standard proxies resend the accumulated conversation and system instructions under `parent_message_id`, causing quadratic $O(N^2)$ context growth, premature prompt truncation, and token limits.
   * *Fix:* On turn 1, system instructions and schemas are sent. On subsequent turns, only the incremental delta (e.g. `[Tool Result]` or new user turn) is streamed. The model references prior turns directly from the remote session tree.
2. **Resilient Tool-Calling Engine (DSML & XML)**
   * *Unclosed Tag Recovery:* DeepSeek Web frequently terminates generation immediately after `</invoke>`, omitting closing `</tool_calls>`. FreedshAPI safely recovers and extracts the call instead of leaking raw XML to chat.
   * *RFC 8259 Type Coercion:* DeepSeek Web emits parameter values like `<parameter name="offset">1145</parameter>` as raw strings. The proxy auto-coerces numbers, booleans, and nulls into native types, satisfying strict JSON schema validators (`offset must be a number`).
   * *Windows Paths & Collapsed Envelopes:* Flawlessly parses unescaped Windows paths (`C:\Users\...`) and handles mixed JSON/invoke payloads.
3. **SSE First-Token Preservation**
   * DeepSeek Web's Server-Sent Events protocol delivers the initial character/word in the fragment envelope (`response/fragments`) and subsequent characters via delta patches (`response/fragments/-1/content`). FreedshAPI flushes the initial chunk immediately, eliminating dropped first characters ("ратко" → "Кратко", "ет" → "Нет").
4. **Zero-Token Deadlock Self-Healing**
   * If an upstream turn desynchronizes or returns 0 tokens, FreedshAPI automatically resets the remote session to prevent the agent from getting trapped in an infinite retry loop.
5. **Session Isolation & Clean Compaction (`/compact`)**
   * Background utility requests (such as automatic session title generation) are routed to isolated sub-agents and never hijack the active chat.
   * Compaction summarizer prompts are isolated. When the agent resumes with `<compacted-summary>`, FreedshAPI cleanly creates a brand new web chat with only the condensed checkpoint, discarding obsolete history.
6. **Zero External Dependencies**
   * Pure native Node.js (18+). No external npm packages required. Includes a built-in automated test suite with **53 unit tests**.

---

### Quick Start

1. **Clone and Authorize:**
   ```bash
   git clone https://github.com/<your-username>/FreedshAPI.git
   cd FreedshAPI
   npm run auth
   ```
   Choose option `1` to log into your DeepSeek account via the dedicated Chrome profile. Your session cookies will be saved locally to `deepseek-auth.json` (gitignored).

2. **Start the Proxy:**
   ```bash
   npm start
   ```
   The server starts on `http://127.0.0.1:9655`.

3. **Verify Health:**
   ```bash
   curl http://127.0.0.1:9655/health
   ```

---

### Container Deployment (Rootless Podman / Docker)

Run in a secure, rootless, read-only container:
```bash
podman run -d \
  --publish 127.0.0.1:9655:9655 \
  --secret free-deepseek-auth,target=/run/secrets/deepseek-auth.json,mode=0400 \
  --secret free-deepseek-proxy-key,target=/run/secrets/proxy-api-key,mode=0400 \
  --read-only \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  freedsh-api
```

---

### Integrating with Autonomous Agents

#### DeepSeek Harness (`dsh`)
In your `dsh` configuration:
* **Base URL:** `http://127.0.0.1:9655/v1`
* **API Key:** any placeholder string (e.g. `sk-dummy`)
* **Model:** `deepseek-chat` (or `v4-pro` / `deepseek-reasoner`)

#### Open WebUI
* Add an OpenAI connection with URL `http://127.0.0.1:9655/v1` and key `dummy`.

---

### Automated Testing

Run the full automated test suite:
```bash
npm test
```
Outputs:
```text
✔ 53 tests passed
✔ 0 tests failed
```

---

<a name="русский"></a>
## Русский

### Описание

**FreedshAPI** — это легковесный локальный API-шлюз для **DeepSeek Web Chat** (`chat.deepseek.com`). Он позволяет использовать ваш обычный аккаунт DeepSeek (через сохранённую браузерную сессию) как полноценную замену платного API, совместимую со спецификациями OpenAI `/v1/chat/completions`, Anthropic `/v1/messages` и OpenAI Responses.

Обычные веб-прокси справляются с простыми вопросами, но ломаются при подключении автономных агентов разработки (**dsh / DeepSeek Harness**, **Claude Code**, **Aider**). **FreedshAPI** решает эти проблемы за счёт синхронизации дерева сессий, отказоустойчивого парсера инструментов и исправления стриминга.

### Ключевые архитектурные улучшения

1. **Инкрементальная передача контекста (Multi-Turn Delta Sync)**
   * *Проблема:* Агентные клиенты на каждом шаге шлют всю историю заново. Стандартные прокси каждый раз передавали весь накопленный диалог, вызывая квадратичный $O(N^2)$ рост контекста, забивая лимиты веб-чата и приводя к обрезке данных.
   * *Решение:* На первом шаге передаются инструкции и схемы. На последующих шагах прокси отправляет **только дельту** (результат выполнения инструмента `[Tool Result]` или новую реплику). Предыдущий контекст берётся из дерева сессии самого DeepSeek.
2. **Надёжный парсер инструментов (DSML & XML)**
   * *Поддержка незакрытых тегов:* Веб-модель DeepSeek часто завершает вывод сразу после `</invoke>`, опуская `</tool_calls>`. FreedshAPI корректно закрывает и парсит вызов, не давая сырому XML просочиться в чат.
   * *Автоматическое приведение типов:* Значения вроде `<parameter name="offset">1145</parameter>` автоматически преобразуются в числа, булевы значения и null, устраняя ошибки строгой валидации схем в `dsh`.
   * *Пути Windows и сложные аргументы:* Корректная обработка путей Windows (`C:\Users\...`) с обратными слэшами и смешанных JSON-обёрток.
3. **Исправление потери первых символов в стриминге (SSE)**
   * Протокол Server-Sent Events DeepSeek передает первый символ в заголовке фрагмента (`response/fragments`), а последующие — дельта-патчами. FreedshAPI немедленно отправляет стартовый чанк клиенту («ратко» → «Кратко», «ет» → «Нет»).
4. **Защита от зацикливания (Self-Healing)**
   * При получении пустого ответа (0 токенов) прокси автоматически сбрасывает сессию, выводя агента из бесконечного цикла повторов.
5. **Изоляция сессий и чистый сброс при `/compact`**
   * Фоновые запросы (например, генерация названия сессии) изолируются в отдельные суб-сессии и не перехватывают управление основным чатом.
   * При сжатии истории (`/compact`) прокси начинает чистый веб-чат, передавая туда **только** итоговую компактную выжимку.
6. **Ноль внешних зависимостей**
   * Чистый Node.js 18+. Набор тестов из **53 юнит-тестов** встроен прямо в проект.

---

### Быстрый старт

1. **Клонирование и авторизация:**
   ```bash
   git clone https://github.com/<your-username>/FreedshAPI.git
   cd FreedshAPI
   npm run auth
   ```
   Выберите пункт `1` для входа в аккаунт DeepSeek в профиле Chrome. Данные сессии сохранятся локально в `deepseek-auth.json` (файл скрыт в `.gitignore`).

2. **Запуск сервера:**
   ```bash
   npm start
   ```
   Сервер доступен по адресу `http://127.0.0.1:9655`.

3. **Проверка работоспособности:**
   ```bash
   curl http://127.0.0.1:9655/health
   ```

---

### Подключение к `dsh` (DeepSeek Harness)

В настройках `dsh`:
* **URL:** `http://127.0.0.1:9655/v1`
* **API Key:** любой ключ (например, `sk-dummy`)
* **Модель:** `deepseek-chat` (или `deepseek-reasoner` / `v4-pro`)

---

### Запуск тестов

```bash
npm test
```
Все 53 теста выполняются стандартным Node.js test runner без сторонних пакетов.

---

### Лицензия и благодарности

* Проект распространяется под лицензией **MIT License**.
* Основан на оригинальных исследованиях веб-протокола DeepSeek разработчиком **Tajerek** (ForgetMeAI).
* Расширен, оптимизирован и адаптирован для автономных AI-агентов.
