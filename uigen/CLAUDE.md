# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Команды

```bash
npm run setup        # Установить зависимости + сгенерировать Prisma клиент + накатить миграции
npm run dev          # Запустить dev-сервер (Next.js + Turbopack)
npm run build        # Продакшн сборка
npm run lint         # ESLint
npm test             # Запустить все тесты (Vitest + jsdom)
npx vitest run path/to/file.test.ts  # Запустить один тестовый файл
npm run db:reset     # Сбросить и пересоздать SQLite базу данных
```

Все npm-скрипты уже включают `NODE_OPTIONS='--require ./node-compat.cjs'` — дополнительных настроек не требуется.

## Архитектура

### Общее описание
UIGen — AI-генератор React-компонентов. Пользователь описывает компонент в чате, Claude генерирует код через вызовы инструментов, результат хранится в виртуальной файловой системе в памяти, iframe отрисовывает живой предпросмотр.

### Ключевые потоки данных

1. **Чат → AI → VFS → Предпросмотр**
   - Пользователь отправляет сообщение через `ChatInterface` → Vercel AI SDK стримит ответ из `/api/chat`
   - API-роут (`src/app/api/chat/route.ts`) вызывает `streamText` с двумя инструментами: `str_replace_editor` и `file_manager`
   - Вызовы инструментов стримятся обратно на клиент, где `FileSystemContext.handleToolCall` применяет их к in-memory `VirtualFileSystem`
   - `PreviewFrame` перерисовывает iframe при каждом изменении `refreshTrigger`

2. **Виртуальная файловая система** (`src/lib/file-system.ts`)
   - `VirtualFileSystem` — класс, хранящий дерево `Map<string, FileNode>`
   - Существует только в памяти клиента — файлы на диск не записываются
   - Сериализуется в `Record<string, FileNode>` для передачи в API-запросах и сохранения в БД
   - Синглтон `fileSystem` используется на стороне сервера в API-роуте; клиентские экземпляры живут внутри React-контекста

3. **Пайплайн предпросмотра** (`src/lib/transform/jsx-transformer.ts`)
   - Трансформирует JSX/TSX в JS прямо в браузере через `@babel/standalone`
   - Строит [import map](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script/type/importmap) с blob URL для локальных файлов и `esm.sh` URL для сторонних пакетов
   - Инжектирует import map и React root в iframe через `srcdoc` с атрибутом `allow-scripts allow-same-origin`

4. **Аутентификация** (`src/lib/auth.ts`, `src/middleware.ts`)
   - JWT в `httpOnly` cookie (`auth-token`), срок жизни 7 дней
   - `getSession()` — только на сервере (использует `next/headers`); `verifySession()` работает в middleware
   - Защищены только `/api/projects` и `/api/filesystem`; `/api/chat` открыт

5. **Персистентность** (`prisma/schema.prisma`)
   - SQLite через Prisma, клиент генерируется в `src/generated/prisma/`
   - `Project.messages` — сериализованная история чата (JSON-строка)
   - `Project.data` — сериализованный снимок VFS (JSON-строка)
   - Проекты опциональны — анонимные сессии работают без сохранения

### Провайдеры контекста
`FileSystemContext` оборачивает экземпляр `VirtualFileSystem` и предоставляет React-мутации, которые также инкрементируют счётчик `refreshTrigger` (вызывая перерисовку `PreviewFrame`). `ChatContext` оборачивает `useChat` из Vercel AI SDK, пробрасывая вызовы инструментов в `FileSystemContext.handleToolCall`.

### Работа без API-ключа
Если `ANTHROPIC_API_KEY` не задан, `src/lib/provider.ts` возвращает mock-провайдер. API-роут определяет это и ограничивает `maxSteps` до 4, чтобы избежать повторений.
