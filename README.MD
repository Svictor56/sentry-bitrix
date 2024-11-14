# Настройка Monolog для записи в файл событий от Sentry для Битрикс

Для того чтобы настроить **Monolog** для записи событий, которые приходят от **Sentry**, в **Битрикс**, вам нужно будет сделать несколько шагов.

### Что нам нужно сделать:

1. Установить **Monolog** через Composer.
2. Установить обработчик для **Sentry** с использованием Monolog.
3. Записать события от **Sentry** в файл с помощью **Monolog**.
4. Интегрировать обработчик с **Битрикс**.

---

### 1. Установка Monolog через Composer

Если у вас еще не установлен **Monolog**, установите его через Composer:

`
composer require monolog/monolog
`

---

### 2. Установка обработчика для Sentry

Чтобы отправлять события в **Sentry**, вы должны установить **Sentry SDK** для PHP:

`
composer require sentry/sdk
`

Этот пакет позволяет работать с **Sentry** и отправлять ошибки и события.

---

### 3. Конфигурация Monolog для записи в файл с событиями от Sentry

В этом примере мы будем использовать **Monolog** для записи ошибок и исключений, которые приходят от **Sentry**, в файл.

#### Пример конфигурации для записи событий от Sentry в файл:

```php
<?php
// Подключаем автозагрузку через Composer
require_once $_SERVER['DOCUMENT_ROOT'] . '/vendor/autoload.php';

// Подключаем нужные классы
use Monolog\Logger;
use Monolog\Handler\StreamHandler;
use Sentry\Monolog\Handler as SentryHandler;
use Sentry\ClientBuilder;
use Monolog\Handler\RotatingFileHandler;

// Настройка Sentry SDK
$dsn = 'https://<PUBLIC_KEY>@sentry.io/<PROJECT_ID>'; // Замените на свой DSN
$client = ClientBuilder::create()->setDsn($dsn)->getClient();

// Создание экземпляра логгера Monolog
$logger = new Logger('bitrix_sentry_logger');

// Настройка пути для хранения логов
$logFilePath = $_SERVER['DOCUMENT_ROOT'] . '/logs/sentry_logs.log';

// Добавляем обработчик для записи логов в файл
$logger->pushHandler(new RotatingFileHandler($logFilePath, 7, Logger::ERROR)); // С ротацией (максимум 7 файлов)

// Добавляем обработчик для отправки событий в Sentry
$logger->pushHandler(new SentryHandler($client));

// Пример использования логирования

try {
    // Пример исключения, которое будет логироваться
    throw new Exception('Это пример ошибки, которая будет отправлена в Sentry.');
} catch (Exception $e) {
    // Логируем ошибку в файл и отправляем в Sentry
    $logger->error('Ошибка: ' . $e->getMessage(), ['exception' => $e]);
}

// Логирование обычных сообщений
$logger->info('Это обычное информационное сообщение, которое также будет записано в файл.');
```

### Пояснение:

- **SentryHandler**: Это специальный обработчик для **Monolog**, который отправляет сообщения в **Sentry**. Мы используем его для отправки логов в **Sentry**.
- **RotatingFileHandler**: Мы добавляем ротацию для логов, чтобы старые логи удалялись после того, как их накопится слишком много. В этом примере ротация настроена на 7 файлов.
- **$logger->error()**: Мы логируем ошибку, которую отправляем в **Sentry**, а также записываем ее в файл.
- **$dsn**: Это ваш **DSN** из **Sentry**, который позволяет отправлять события в ваш проект на Sentry.

---

### 4. Интеграция в Битрикс

В **Битрикс** вы можете интегрировать этот код в нужное место, например, в обработчик ошибок или в любую другую логику, где необходимо логировать ошибки и отправлять их в **Sentry**.

Пример интеграции с Битрикс:

`
use Bitrix\Main\EventManager;
use Monolog\Logger;
use Monolog\Handler\StreamHandler;
use Sentry\Monolog\Handler as SentryHandler;
use Sentry\ClientBuilder;

// Настройка Sentry
$dsn = 'https://<PUBLIC_KEY>@sentry.io/<PROJECT_ID>';
$client = ClientBuilder::create()->setDsn($dsn)->getClient();

// Создание логгера
$logger = new Logger('bitrix_sentry_logger');
$logFilePath = $_SERVER['DOCUMENT_ROOT'] . '/logs/sentry_logs.log';
$logger->pushHandler(new StreamHandler($logFilePath, Logger::ERROR));
$logger->pushHandler(new SentryHandler($client));

// Добавляем обработчик на событие завершения страницы (OnEpilog)
EventManager::getInstance()->addEventHandler("main", "OnEpilog", function() use ($logger) {
    global $APPLICATION;

    // Если есть ошибка, записываем её в файл и отправляем в Sentry
    if ($APPLICATION->GetException()) {
        $logger->error('Ошибка: ' . $APPLICATION->GetException()->GetString(), ['exception' => $APPLICATION->GetException()]);
    }
});
`

### Пояснение:

- Мы добавили обработчик события **OnEpilog**, который будет срабатывать в конце работы скрипта. Этот обработчик проверяет, если в Битриксе произошла ошибка (через `$APPLICATION->GetException()`), то ошибка будет записана в файл и отправлена в **Sentry**.

---

### 5. Ротация логов

Чтобы избежать бесконечного роста файлов логов, мы использовали **RotatingFileHandler**. Этот обработчик автоматически удаляет старые логи, когда их количество превышает 7 файлов. Вы можете изменить это количество или настройки, как вам нужно.

`
$logger->pushHandler(new RotatingFileHandler($logFilePath, 7, Logger::ERROR, true, 10485760));
`

Этот код будет создавать не более 7 файлов, каждый из которых будет иметь размер не более 10 МБ, после чего старые логи будут удаляться.

---

### Заключение

- Мы настроили **Monolog** для записи логов как в файл, так и в **Sentry**.
- Для этого мы использовали **RotatingFileHandler** для управления размером логов.
- Мы интегрировали обработку ошибок Битрикс с **Monolog** и **Sentry**, чтобы автоматически записывать ошибки и исключения в файл и отправлять их в **Sentry**.
- Теперь, когда в Битриксе происходит ошибка, она записывается в файл и отправляется в **Sentry**, обеспечивая централизованное логирование и мониторинг.

Эта настройка позволяет вам удобно отслеживать ошибки и события, связанные с вашим приложением, с сохранением локальных логов и интеграцией с **Sentry** для централизованного мониторинга.
