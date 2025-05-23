# Бэкенд Приложения RealTone (Анализатор Аудио)

## 1. Обзор Архитектуры Бэкенда

Бэкенд-система RealTone построена на микросервисной архитектуре, где основным координатором выступает **Go REST API**, а задачи по анализу аудио выполняет специализированный **Python gRPC сервис**.

**Ключевые компоненты бэкенда:**

1.  **Go REST API (`go_web/`):**
    *   Написан на Go с использованием фреймворка Gin Gonic.
    *   Предоставляет HTTP/S эндпоинты для клиентских приложений (фронтенд).
    *   Отвечает за аутентификацию пользователей, управление метаданными, загрузку файлов в S3-совместимое хранилище и оркестрацию процесса анализа аудио.
2.  **Python gRPC Сервис (`server/`):**
    *   Реализован на Python с использованием grpcio.
    *   Предоставляет gRPC эндпоинт для анализа аудио.
    *   Взаимодействует с S3-хранилищем для получения аудиоданных по ключу.
    *   Использует модель машинного обучения (на базе PyTorch, WavLM) для детекции аудио-спуфинга.
    *   Применяет Redis для кэширования результатов анализа отдельных аудиочанков.
3.  **PostgreSQL:**
    *   Реляционная база данных для хранения информации о пользователях и метаданных загруженных аудиофайлов.
4.  **MinIO (S3-совместимое хранилище):**
    *   Объектное хранилище для сохранения оригинальных аудиофайлов, загруженных пользователями.
5.  **Redis:**
    *   In-memory хранилище ключ-значение, используемое Python gRPC сервисом для кэширования обработанных аудиочанков и их предсказаний.

**Взаимодействие сервисов:**

*   Клиент (фронтенд) взаимодействует с Go REST API.
*   Go REST API при загрузке аудиофайла сохраняет его в MinIO, записывает метаданные в PostgreSQL, а затем вызывает Python gRPC сервис для анализа.
*   Python gRPC сервис получает из запроса от Go API информацию о местонахождении файла в MinIO, скачивает его, обрабатывает и возвращает результат анализа обратно в Go API.
*   Go API возвращает агрегированный ответ (включая результаты анализа) клиенту.

## 2. Docker и Оркестрация (`go_web/docker-compose.yml`)

Для упрощения развертывания и управления зависимостями используется Docker Compose.

**Конфигурация `docker-compose.yml`:**

*   **Сервисы:**
    *   `go_app`: Собирается из `Dockerfile` в директории `go_web/`. Запускает Go REST API.
        *   Зависит от `postgres_db` и `minio`.
        *   Пробрасывает порт, указанный в `${GO_APP_PORT}` (по умолчанию 8080).
        *   Использует `.env` файл для переменных окружения.
        *   Переменная `DB_HOST` явно установлена в `postgres_db` для корректного подключения внутри Docker-сети.
    *   `postgres_db`: Использует официальный образ `postgres:15-alpine`.
        *   Настраивается через переменные окружения из `.env` (`DB_USER`, `DB_PASSWORD`, `DB_NAME`).
        *   Пробрасывает порт PostgreSQL (`${DB_PORT}` или 5432).
        *   Использует volume `postgres_data` для персистентности данных.
        *   Автоматически выполняет скрипт `go_web/scripts/init.sql` при первом запуске для создания таблиц.
    *   `minio`: Использует образ `quay.io/minio/minio:latest`.
        *   Настраивается через переменные окружения из `.env` (`S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`).
        *   Пробрасывает API порт MinIO (`${MINIO_API_PORT}` или 9000) и порт консоли (`${MINIO_CONSOLE_PORT}` или 9001).
        *   Использует volume `minio_data` для персистентности данных.
        *   Запускает MinIO сервер с указанием хранить данные в `/data` и активирует консоль.
    *   `redis_cache`: Использует официальный образ `redis:7-alpine`.
        *   Пробрасывает стандартный порт Redis (6379).
        *   Использует volume `redis_data` для персистентности данных.
*   **Сеть:**
    *   Все сервисы объединены в одну bridge-сеть `app_network`, что позволяет им взаимодействовать друг с другом по именам сервисов (например, `go_app` может обращаться к `postgres_db` по имени хоста `postgres_db`).
*   **Volumes:**
    *   `postgres_data`, `minio_data`, `redis_data`: Именованные volumes для обеспечения сохранности данных при перезапуске или удалении контейнеров.

**Важное замечание:** Python gRPC сервис (`server/`) **не включен** в данный `docker-compose.yml`. Это означает, что при запуске Docker-композиции Go-приложение будет пытаться подключиться к Python-сервису по адресу, указанному в `PYTHON_GRPC_SERVICE_ADDR` (по умолчанию `localhost:50052` или `host.docker.internal:50052` для Windows/Mac, если Python запускается локально на хост-машине). Для полноценного Docker-окружения Python-сервис также должен быть добавлен в `docker-compose.yml`.

## 3. Go REST API (`go_web/`)

**Ключевые компоненты и логика:**

*   **Точка входа:** `cmd/server/main.go`
    *   Загрузка конфигурации (`internal/config/config.go`) из переменных окружения (и `.env` файла).
    *   Инициализация логгера (Zap).
    *   Подключение к PostgreSQL (`internal/database/database.go`).
    *   Инициализация S3-сервиса (`internal/s3service/s3service.go`) для работы с MinIO, включая проверку/создание бакета.
    *   Инициализация gRPC-клиента (`internal/grpc_clients/python_audio_analyzer_client.go`) для взаимодействия с Python-сервисом анализа аудио.
    *   Настройка роутера Gin Gonic, включая CORS middleware.
    *   Инициализация репозиториев для работы с БД.
    *   Инициализация сервиса аутентификации (`internal/auth/auth.go`).
    *   Регистрация хэндлеров (`internal/handlers/`) для обработки HTTP-запросов.
*   **Конфигурация (`internal/config/config.go`):**
    *   Загружает параметры для подключения к БД, JWT, S3 (MinIO), адрес Python gRPC сервиса и настройки логгирования из переменных окружения.
*   **Обработчики (`internal/handlers/`):
    *   `user_handler.go`: Обработка запросов `/api/v1/users/register` и `/api/v1/users/login`.
    *   `audio_handler.go`: Обработка запроса `/api/v1/audio/upload` (защищено JWT middleware).
        1.  **Аутентификация:** Проверка JWT токена пользователя.
        2.  **Валидация файла:** Проверка размера файла (до 10MB) и расширения (WAV, MP3, OGG).
        3.  **Загрузка в MinIO:**
            *   Генерация уникального ключа (`s3Key`) для файла в формате `userID/timestamp/filename.ext`.
            *   Вызов `s3Service.UploadFile` для загрузки файла в бакет MinIO (имя бакета из конфига).
        4.  **Сохранение метаданных в PostgreSQL:**
            *   Создание записи в таблице `audio_files` с информацией о файле (ID, userID, s3Key, имя, тип, размер).
        5.  **Вызов Python gRPC сервиса:**
            *   Создание контекста с таймаутом для gRPC вызова.
            *   Вызов метода `pyAnalyzerClient.AnalyzeAudio`, передавая имя бакета MinIO и `s3Key` файла.
        6.  **Формирование ответа клиенту:**
            *   Создание структуры `UploadAndAnalyzeAudioResponse`, включающей `FileID`, `S3Key`, `FileURL`, а также `AnalysisResults` (преобразованные из ответа gRPC) и возможные `AnalysisError`.
            *   Отправка JSON-ответа клиенту (фронтенду) со статусом `http.StatusCreated`.
*   **gRPC Клиент (`internal/grpc_clients/python_audio_analyzer_client.go`):
    *   Реализует интерфейс `PythonAudioAnalyzerClient`.
    *   Функция `NewPythonAudioAnalyzerClient` устанавливает соединение с Python gRPC сервисом по адресу из конфигурации (используя небезопасные учетные данные и блокирующее подключение при инициализации).
    *   Метод `AnalyzeAudio` формирует `AnalyzeAudioRequest` (с `MinioBucketName` и `MinioObjectKey`), выполняет gRPC вызов к Python-сервису и возвращает `AnalyzeAudioResponse`.
*   **Middleware (`internal/middleware/auth_middleware.go`):
    *   Проверяет наличие и валидность JWT токена в `Authorization` заголовке.
    *   Извлекает `claims` (включая `UserID`) и добавляет их в контекст Gin для использования в последующих обработчиках.

## 4. Python gRPC Сервис (`server/`)

**Ключевые компоненты и логика (`server/grpc_server.py`):**

*   **gRPC Серверная Реализация:**
    *   Класс `AudioAnalysisServicer` наследуется от сгенерированного `audio_analyzer_pb2_grpc.AudioAnalysisServicer`.
    *   При инициализации (`__init__`):
        *   Определяет устройство для PyTorch (`cuda` или `cpu`).
        *   Загружает предварительно обученную модель для анализа аудио из чекпоинта (`chk3.pth`) с помощью функции `load_model_from_checkpoint` из `inference.py`.
        *   Инициализирует клиент Redis, подключаясь к хосту и порту из переменных окружения (`REDIS_HOST`, `REDIS_PORT`).
        *   Инициализирует клиент MinIO для доступа к объектному хранилищу (`MINIO_ENDPOINT`, `MINIO_ACCESS_KEY`, и т.д.).
*   **Метод `AnalyzeAudio`:**
    1.  **Получение запроса:** Принимает `AnalyzeAudioRequest` с `minio_bucket_name` и `minio_object_key`.
    2.  **Проверки:** Валидирует доступность Redis и MinIO клиента.
    3.  **Скачивание файла из MinIO:**
        *   Использует `minio_client.get_object` для получения содержимого аудиофайла из указанного бакета и по указанному ключу.
    4.  **Предобработка аудио:**
        *   Загружает аудиоданные из байтового потока с помощью `torchaudio.load`.
        *   При необходимости выполняет ресемплинг до `SAMPLE_RATE` (16000 Гц).
        *   Нормализует сигнал.
    5.  **Нарезка на чанки и сохранение в Redis:**
        *   Аудиосигнал делится на чанки размером `NUM_SAMPLES` (64000, что соответствует 4 секундам при 16000 Гц).
        *   Каждый чанк (как массив numpy float32) сохраняется в Redis под ключом вида `internal_request_id:chunk_{idx}` с временем жизни `REDIS_CHUNK_EXPIRY_SECONDS`.
    6.  **Обработка чанков (может быть распараллелена в будущем, сейчас последовательно):**
        *   Для каждого индекса чанка вызывается `_process_chunk_from_redis`.
        *   `_process_chunk_from_redis`:
            *   Извлекает данные чанка из Redis.
            *   Преобразует байты обратно в тензор PyTorch.
            *   Вызывает `_predict_score_for_chunk_tensor` для получения оценки от модели.
            *   `_predict_score_for_chunk_tensor`: выполняет инференс модели (`self.model(chunk_tensor)`) и применяет сигмоиду для получения вероятности (0-1).
            *   Формирует объект `AudioChunkPrediction` с `chunk_id`, округленным `score`, `start_time_seconds` и `end_time_seconds`.
    7.  **Формирование ответа:**
        *   Собирает все `AudioChunkPrediction` в список `predictions`.
        *   Если были ошибки на каком-либо этапе, заполняет `error_message`.
        *   Возвращает `AnalyzeAudioResponse`.
*   **Конфигурация:** Большая часть конфигурации (адреса Redis, MinIO, пути к модели) получается из переменных окружения.

## 5. База Данных PostgreSQL (`go_web/scripts/init.sql`)

*   **Схема:**
    *   **`users` таблица:**
        *   `id` (UUID, PK): Уникальный идентификатор пользователя.
        *   `username` (VARCHAR, UNIQUE): Имя пользователя.
        *   `email` (VARCHAR, UNIQUE): Email пользователя.
        *   `password_hash` (VARCHAR): Хэш пароля.
        *   `created_at`, `updated_at` (TIMESTAMP WITH TIME ZONE): Временные метки.
        *   Триггер `trigger_users_updated_at` автоматически обновляет `updated_at` при изменении записи.
    *   **`audio_files` таблица:**
        *   `id` (UUID, PK): Уникальный идентификатор аудиозаписи.
        *   `user_id` (UUID, FK -> users.id): Идентификатор пользователя, загрузившего файл (с `ON DELETE CASCADE`).
        *   `s3_key` (VARCHAR, UNIQUE): Ключ объекта в S3/MinIO.
        *   `original_filename` (VARCHAR): Оригинальное имя файла.
        *   `content_type` (VARCHAR): MIME-тип файла.
        *   `size_bytes` (BIGINT): Размер файла в байтах.
        *   `uploaded_at` (TIMESTAMP WITH TIME ZONE): Время загрузки.
    *   Индексы созданы для `users.email`, `audio_files.user_id`, `audio_files.s3_key` для ускорения поиска.
*   **Использование:** Go REST API использует PostgreSQL для:
    *   Хранения учетных данных пользователей и их профилей.
    *   Сохранения метаинформации о загруженных аудиофайлах, связывая их с пользователями и их расположением в MinIO.

## 6. Хранилище Объектов MinIO

*   **Назначение:** Используется как S3-совместимое хранилище для персистентного хранения оригинальных аудиофайлов, загружаемых пользователями.
*   **Взаимодействие:**
    *   **Go REST API:** Загружает файлы в MinIO после получения их от клиента. Сохраняет `s3_key` в PostgreSQL. Передает `s3_key` и имя бакета Python-сервису.
    *   **Python gRPC Сервис:** Использует `s3_key` и имя бакета для скачивания файла из MinIO для последующего анализа.
*   **Конфигурация (в `docker-compose.yml` и Go/Python коде):** Эндпоинт, ключи доступа, имя бакета.

## 7. Кэш Redis

*   **Назначение:** Используется Python gRPC сервисом для временного хранения (кэширования) отдельных аудиочанков после их нарезки из исходного файла.
*   **Взаимодействие:**
    *   **Python gRPC Сервис:**
        1.  После скачивания файла из MinIO и его нарезки на чанки, каждый чанк (как `float32 numpy array` в виде байтов) сохраняется в Redis с ключом, включающим уникальный идентификатор запроса и индекс чанка (`request_id:chunk_{idx}`).
        2.  При обработке (инференсе) каждого чанка, он сначала извлекается из Redis по ключу.
        3.  Устанавливается время жизни для ключей в Redis (`REDIS_CHUNK_EXPIRY_SECONDS`), чтобы старые данные автоматически удалялись.
*   **Преимущества:**
    *   Может уменьшить повторную обработку одних и тех же данных, если бы такая логика была.
    *   В текущей реализации, основное применение - это передача данных между этапом нарезки и этапом обработки каждого чанка, что позволяет потенциально распараллелить обработку чанков в будущем (например, с помощью `ThreadPoolExecutor` или Celery воркеров, которые бы читали из Redis).

## 8. gRPC Определения (`proto/audio_analyzer.proto`)

Определяет контракт для межсервисного взаимодействия между Go API и Python-сервисом:

*   **Сервис `AudioAnalysis`:**
    *   Метод `AnalyzeAudio (AnalyzeAudioRequest) returns (AnalyzeAudioResponse)`.
*   **Сообщение `AnalyzeAudioRequest`:**
    *   `minio_bucket_name (string)`: Название бакета в MinIO.
    *   `minio_object_key (string)`: Ключ (путь) к файлу в MinIO.
*   **Сообщение `AudioChunkPrediction`:**
    *   `chunk_id (string)`: Идентификатор чанка.
    *   `score (float)`: Вероятность спуфинга (0-1).
    *   `start_time_seconds (float)`: Время начала чанка.
    *   `end_time_seconds (float)`: Время окончания чанка.
*   **Сообщение `AnalyzeAudioResponse`:**
    *   `predictions (repeated AudioChunkPrediction)`: Список предсказаний по чанкам.
    *   `error_message (string)`: Сообщение об ошибке, если возникла.
*   Опция `go_package` используется для указания пути генерации Go-стабов.

## 9. Ключевые Элементы и Потоки Данных

1.  **Загрузка и Первичная Обработка (Go API):**
    *   Фронтенд -> Go API (`/api/v1/audio/upload` с файлом и JWT).
    *   Go API: Аутентификация -> Валидация -> MinIO (сохранение файла) -> PostgreSQL (сохранение метаданных).
2.  **Запрос на Анализ (Go API -> Python gRPC):**
    *   Go API (gRPC клиент) -> Python gRPC Сервис (`AnalyzeAudio` метод с `bucket_name`, `object_key`).
3.  **Анализ Аудио (Python gRPC Сервис):**
    *   Python Сервис: MinIO (чтение файла) -> Нарезка на чанки -> Redis (сохранение чанков).
    *   Python Сервис: Redis (чтение чанка) -> Модель ML (инференс) -> Формирование `AudioChunkPrediction`.
4.  **Возврат Результатов Анализа (Python gRPC -> Go API -> Фронтенд):**
    *   Python gRPC Сервис -> Go API (gRPC клиент) (ответ с `predictions` и/или `error_message`).
    *   Go API -> Фронтенд (JSON ответ с результатами, готовыми для отображения).

Этот документ охватывает основные аспекты бэкенд-системы вашего проекта.
