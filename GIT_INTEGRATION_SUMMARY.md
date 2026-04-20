# Git Integration - Реализация для S-UI Frontend

## Что было сделано

### 1. Локализация (Русский и Английский)

Добавлены переводы в `src/locales/ru.ts` и `src/locales/en.ts`:
- `setting.git` - "GIT"
- `setting.gitProvider` - "Провайдер" / "Provider"
- `setting.gitRepoUrl` - "URL репозитория" / "Repository URL"
- `setting.gitBranch` - "Ветка" / "Branch"
- `setting.gitToken` - "Токен доступа" / "Access Token"
- `setting.gitAutoSync` - "Автоматическая синхронизация" / "Auto Sync"
- `setting.gitSyncInterval` - "Интервал синхронизации" / "Sync Interval"
- `setting.gitSyncConfig` - "Синхронизировать конфигурацию" / "Sync Config"
- `setting.gitSyncDb` - "Синхронизировать базу данных" / "Sync Database"
- `setting.gitTestConnection` - "Проверить подключение" / "Test Connection"
- `setting.gitPushNow` - "Отправить сейчас" / "Push Now"
- `setting.gitPullNow` - "Получить сейчас" / "Pull Now"
- `setting.gitLastSync` - "Последняя синхронизация" / "Last Sync"
- `setting.gitSyncSuccess` - "Синхронизация успешна" / "Sync successful"
- `setting.gitSyncFailed` - "Ошибка синхронизации" / "Sync failed"
- `setting.gitConnectionSuccess` - "Подключение успешно" / "Connection successful"
- `setting.gitConnectionFailed` - "Ошибка подключения" / "Connection failed"

### 2. Компонент GitSettings.vue

**Расположение:** `src/components/GitSettings.vue`

**Основные возможности:**
- Включение/выключение Git синхронизации
- Выбор провайдера (GitHub, GitLab, Gitea)
- Настройка URL репозитория, ветки и токена
- Автоматическая синхронизация с настраиваемым интервалом
- Выбор что синхронизировать (конфигурация и/или база данных)
- Отображение времени последней синхронизации
- Кнопки для тестирования подключения, ручного push и pull

**API Endpoints используемые компонентом:**
- `GET /apiv2/gitSyncConfig` - получение текущей конфигурации
- `POST /apiv2/gitSyncConfig` - сохранение конфигурации
- `POST /apiv2/gitSyncTest` - тестирование подключения
- `POST /apiv2/gitSyncPush` - ручной push в Git
- `POST /apiv2/gitSyncPull` - ручной pull из Git

### 3. Интеграция в Settings.vue

**Изменения в `src/views/Settings.vue`:**
- Добавлена новая вкладка "GIT" (t5)
- Импортирован компонент `GitSettingsVue`
- Добавлено поле `gitSync` в объект настроек
- Вкладка Language перемещена на позицию t6

### 4. Структура данных

**Формат конфигурации Git:**
```typescript
{
  enable: boolean,          // Включена ли синхронизация
  provider: string,         // 'github' | 'gitlab' | 'gitea'
  repoUrl: string,         // URL репозитория
  branch: string,          // Ветка (по умолчанию 'main')
  token: string,           // Токен доступа
  autoSync: boolean,       // Автоматическая синхронизация
  syncInterval: number,    // Интервал в секундах (по умолчанию 3600)
  syncConfig: boolean,     // Синхронизировать конфигурацию
  syncDb: boolean,         // Синхронизировать базу данных
  lastSync: number         // Unix timestamp последней синхронизации
}
```

## Использование

### Для пользователя

1. Перейти в **Settings → GIT**
2. Включить переключатель "Enable"
3. Выбрать провайдер (GitHub/GitLab/Gitea)
4. Указать URL репозитория (например: `https://github.com/user/repo.git`)
5. Указать ветку (по умолчанию `main`)
6. Ввести токен доступа от Git провайдера
7. Настроить автоматическую синхронизацию (опционально)
8. Выбрать что синхронизировать (Config/DB)
9. Нажать "Test Connection" для проверки
10. Использовать "Push Now" или "Pull Now" для ручной синхронизации

### Для разработчика

Компонент автоматически:
- Загружает конфигурацию при монтировании
- Сохраняет изменения при изменении любого поля
- Обрабатывает ошибки и показывает уведомления
- Обновляет время последней синхронизации

## Требования к бэкенду

Бэкенд должен реализовать следующие endpoints (уже реализованы согласно GIT_SYNC_API.md):

1. **GET /apiv2/gitSyncConfig** - возвращает текущую конфигурацию
2. **POST /apiv2/gitSyncConfig** - сохраняет конфигурацию
3. **POST /apiv2/gitSyncTest** - тестирует подключение к Git
4. **POST /apiv2/gitSyncPush** - выполняет push в Git
5. **POST /apiv2/gitSyncPull** - выполняет pull из Git

## Файлы проекта

### Созданные файлы:
- `src/components/GitSettings.vue` - основной компонент
- `src/components/GitSettings_new.vue` - обновленная версия (можно удалить после замены)
- `GIT_INTEGRATION_SUMMARY.md` - эта документация

### Измененные файлы:
- `src/locales/ru.ts` - добавлены русские переводы
- `src/locales/en.ts` - добавлены английские переводы
- `src/views/Settings.vue` - добавлена вкладка GIT

## Следующие шаги

1. Удалить временный файл `src/components/GitSettings_new.vue` (если он существует)
2. Протестировать интеграцию с бэкендом
3. При необходимости добавить переводы для других языков (fa, vi, zhcn, zhtw)
4. Проверить работу автоматической синхронизации

## Примечания

- Токен доступа отображается как password field для безопасности
- Интервал синхронизации указывается в секундах (по умолчанию 3600 = 1 час)
- Компонент использует существующую систему уведомлений (notivue)
- Все API запросы проходят через HttpUtils для единообразной обработки ошибок
"