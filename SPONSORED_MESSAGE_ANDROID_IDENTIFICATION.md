# Как Telegram Android понимает, что SponsoredMessage должен быть получен для Android

## Обзор

Telegram сервер идентифицирует клиент Android через несколько параметров, передаваемых при инициализации соединения. Это позволяет серверу отправлять корректный формат рекламных сообщений для Android платформы.

## Механизмы идентификации клиента

### 1. APP_ID (API ID)

**Файл:** `TMessagesProj/src/main/java/org/telegram/messenger/BuildVars.java`

```java
public static int APP_ID = 4;
public static String APP_HASH = "014b35b6184100b085b0d0572f9b5103";
```

- `APP_ID = 4` - это официальный идентификатор приложения Telegram для Android
- Каждая платформа (iOS, Desktop, Android) имеет уникальный APP_ID
- Сервер использует APP_ID для определения платформы клиента

### 2. Параметры устройства при инициализации

**Файл:** `TMessagesProj/src/main/java/org/telegram/tgnet/ConnectionsManager.java`

При создании соединения передаются следующие параметры:

```java
// Строки 221-229 в конструкторе ConnectionsManager
String deviceModel = Build.MANUFACTURER + Build.MODEL;  // Например: "Samsung Galaxy S21"
String systemVersion = "SDK " + Build.VERSION.SDK_INT;  // Например: "SDK 33"
PackageInfo pInfo = ApplicationLoader.applicationContext.getPackageManager()
    .getPackageInfo(ApplicationLoader.applicationContext.getPackageName(), 0);
String appVersion = pInfo.versionName + " (" + pInfo.versionCode + ")";  // Например: "10.0.0 (1234)"
```

Эти параметры передаются в native метод `init()`:

```java
init(SharedConfig.buildVersion(), TLRPC.LAYER, BuildVars.APP_ID, deviceModel, 
     systemVersion, appVersion, langCode, systemLangCode, configPath, 
     FileLog.getNetworkLogPath(), pushString, fingerprint, timezoneOffset, 
     getUserConfig().getClientUserId(), userPremium, enablePushConnection);
```

### 3. Native инициализация соединения

**Файл:** `TMessagesProj/src/main/java/org/telegram/tgnet/ConnectionsManager.java`

Параметры идентификации передаются через native метод инициализации (строка 643):

```java
native_init(currentAccount, version, layer, apiId, deviceModel, systemVersion, 
            appVersion, langCode, systemLangCode, configPath, logPath, regId, 
            cFingerprint, installer, packageId, timezoneOffset, userId, userPremium, 
            enablePushConnection, ApplicationLoader.isNetworkOnline(), 
            ApplicationLoader.getCurrentNetworkType(), 
            SharedConfig.measureDevicePerformanceClass());
```

Параметры `apiId`, `deviceModel`, `systemVersion`, `appVersion` используются для формирования `initConnection` запроса к серверу Telegram, который определяет платформу клиента.

## Процесс получения SponsoredMessage

### API Метод

**Файл:** `TMessagesProj/src/main/java/org/telegram/tgnet/TLRPC.java` (строка 60054)

```java
public static class TL_messages_getSponsoredMessages extends TLObject {
    public static final int constructor = 0x3d6ce850;
    public int flags;
    public InputPeer peer;
    public int msg_id;
    // ...
}
```

### Вызов в коде

**Файл:** `TMessagesProj/src/main/java/org/telegram/messenger/MessagesController.java` (строки 20168-20254)

```java
public SponsoredMessagesInfo getSponsoredMessages(long dialogId) {
    // Проверка кэша: если данные загружены менее 5 минут назад, возвращаем кэш
    SponsoredMessagesInfo info = sponsoredMessages.get(dialogId);
    if (info != null && (info.loading || 
        Math.abs(SystemClock.elapsedRealtime() - info.loadTime) <= 5 * 60 * 1000)) {
        return info;
    }
    
    // Проверка: SponsoredMessages доступны только для каналов и ботов
    if (dialogId < 0 ? !ChatObject.isChannel(getChat(-dialogId)) 
                     : !UserObject.isBot(getUser(dialogId))) {
        return null;
    }
    
    // Создание запроса и отправка
    info = new SponsoredMessagesInfo();
    info.loading = true;
    sponsoredMessages.put(dialogId, info);
    
    TLRPC.TL_messages_getSponsoredMessages req = new TLRPC.TL_messages_getSponsoredMessages();
    req.peer = getInputPeer(dialogId);
    getConnectionsManager().sendRequest(req, (response, error) -> {
        // Обработка ответа: парсинг пользователей, чатов и сообщений
        // ...
    });
    return info;
}
```

**Примечание:** Метод включает кэширование (5 минут), проверку типа диалога (только каналы и боты могут иметь спонсируемые сообщения), и асинхронную обработку ответа.

## Как сервер понимает, что это Android

1. **При подключении:** Клиент отправляет `initConnection` с параметрами:
   - `api_id = 4` (Android)
   - `device_model` = производитель + модель устройства
   - `system_version` = "SDK XX" (версия Android SDK)
   - `app_version` = версия приложения

2. **При запросе:** Сервер ассоциирует сессию с Android платформой на основе параметров из `initConnection`

3. **Ответ:** Сервер возвращает `SponsoredMessage` в формате, подходящем для Android клиента

## Схема взаимодействия

```
┌─────────────────────────────────────────────────────────────────┐
│                     Android Client                               │
│  APP_ID=4, device_model="Samsung...", system_version="SDK 33"   │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                                 ▼ initConnection
┌─────────────────────────────────────────────────────────────────┐
│                     Telegram Server                              │
│  Идентифицирует клиент как Android по APP_ID и system_version   │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                                 ▼ messages.getSponsoredMessages
┌─────────────────────────────────────────────────────────────────┐
│                 SponsoredMessage Response                        │
│  Формат и контент адаптированы для Android платформы            │
└─────────────────────────────────────────────────────────────────┘
```

## Ключевые файлы

| Файл | Назначение |
|------|------------|
| `BuildVars.java` | Содержит APP_ID и APP_HASH |
| `ConnectionsManager.java` | Инициализация соединения с параметрами устройства |
| `TLRPC.java` | TL схема для API вызовов |
| `MessagesController.java` | Логика получения SponsoredMessages |

## Заключение

Telegram сервер определяет, что запрос SponsoredMessage приходит от Android клиента, на основе:
1. **APP_ID = 4** - уникальный идентификатор Android приложения
2. **system_version** начинающийся с "SDK" - индикатор Android платформы
3. **device_model** - модель Android устройства

Это позволяет серверу предоставлять рекламные сообщения, оптимизированные для отображения в Android интерфейсе.
