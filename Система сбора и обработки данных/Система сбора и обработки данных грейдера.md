# Система сбора и обработки данных об использовании онлайн‑редактора кода (грейдера) для Андекс.Университет

## Краткое описание проекта

Автоматизированная система для сбора данных из API грейдера онлайн‑университета, их обработки, загрузки в PostgreSQL и формирования отчётов с отправкой уведомлений. Проект решает задачу мониторинга успеваемости студентов и предоставляет данные для дальнейшего анализа.

**Ключевые технологии:** Python, PostgreSQL, REST API, Google Sheets API, smtplib, ssl, logging, psycopg2, requests, gspread.

## Бизнес‑задача и актуальность

Онлайн‑университет использует грейдер (редактор кода с проверкой решений) внутри обучающей системы (ЛМС). Ежедневно тысячи студентов решают задачи, и данные об их успехах хранятся на стороне грейдера.

**Проблема:** университет не имеет прямого доступа к этим данным для:
* оперативного мониторинга успеваемости;
* анализа эффективности учебных модулей;
* предоставления персонализированной обратной связи студентам.

**Решение:** скрипт на Python, который:
* забирает данные из API грейдера;
* обрабатывает и валидирует их;
* загружает в локальную базу PostgreSQL;
* формирует ежедневные отчёты в Google Sheets;
* отправляет email‑уведомление о завершении работы.

## Цели и задачи

**Цель:** создать автоматизированное решение для сбора и структурирования данных об успеваемости студентов.

**Задачи:**
1. Реализовать запрос к API грейдера для получения данных о попытках студентов.
2. Обработать данные: извлечь поля из `passback_params`, проверить типы и значения.
3. Загрузить данные в PostgreSQL.
4. Настроить логирование работы скрипта с сохранением логов за последние 3 дня.
5. Агрегировать данные за день (количество попыток, успешных решений, уникальных пользователей) и выгрузить в Google Sheets.
6. Настроить отправку email‑уведомлений о завершении работы скрипта.

## Архитектура решения

Система состоит из следующих компонентов:

1. **Скрипт на Python** — основной модуль, выполняющий:
   * запрос к API;
   * обработку и валидацию данных;
   * загрузку в PostgreSQL;
   * агрегацию данных;
   * выгрузку в Google Sheets;
   * отправку email.

2. **База данных PostgreSQL** — хранит сырые данные о попытках студентов:
   ```sql
   CREATE TABLE public.grades (
	  user_id varchar NOT NULL,
	  oauth_consumer_key varchar NULL,
	  lis_result_sourcedid varchar NULL,
	  lis_outcome_service_url varchar NULL,
	  is_correct int NULL,
	  attempt_type varchar NULL,
	  created_at timestamp NULL);

3. **Google Sheets** — таблица с ежедневными агрегированными данными:
* `attempts_total` — общее количество попыток;
* `submits_correct` — количество успешных попыток;
* 'submits_incorrect' — количество неуспешных попыток;
* `DAU` — количество уникальных студентов;
* `runs` — количество запусков.

4. **Логирование** — текстовые файлы с метками времени:
* формат имени: `YYYY-MM-DD.log` (например, `2024-05-20.log`);
* уровни логирования:
  * **INFO** — информационные сообщения о стадиях выполнения (начало/завершение скачивания, загрузки в БД и т. д.);
  * **ERROR** — записи об ошибках (проблемы с подключением к БД, некорректные данные);


5. **Уведомления** — email с отчётом, отправляемый после завершения работы скрипта:
* статус выполнения скрипта (успешно/с ошибками);
* продуктовые метрики;
* ссылки на отчёт в Google Sheets;
* временные метрики.

  ## Реализация
  ### Структура классов
* класс `APIClient` — отвечает за запросы к API и пагинацию;
* класс `DataProcessor` — парсит passback_params, валидирует данные;
* класс `DatabaseManager` — управляет подключением к PostgreSQL и загрузкой данных;
* класс `Logger` — настраивает логирование;
* класс `ReportGenerator` — агрегирует данные и выгружает в Google Sheets;
* класс `NotificationService` — отправляет email‑уведомления.


### Этап 1. Получение данных из API
- Использована библиотека requests для отправки GET‑запросов к endpoint API.
```python
class APIClient:
    def __init__(self, client, client_key: str, api_url: str) -> None:
        self.client = client
        self.client_key = client_key
        self.api_url = api_url

    def fetch_data(self, start, end):
        # Запрос к API, обработка пагинации
        data = {
            'client': self.client,
            'client_key': self.client_key,
            'start': start,
            'end': end
        }
        return requests.get(self.api_url, params=data).json()
```

### Этап 2. Обработка и валидация данных
Распарсен `passback_params` (JSON‑строка внутри ответа API) для извлечения:
- oauth_consumer_key;
- lis_result_sourcedid;
- lis_outcome_service_url.

Валидация типов данных:
is_correct — int;
created_at — datetime.

```python
    def validate_record(self, record: dict):
        types = {
            "is_correct": int,
            "created_at": datetime
        }

        for key, value in record.items():
            if not isinstance(value, types.get(key, str)):
                raise ValueError()
```

### Этап 3. Загрузка в PostgreSQL
- Подключение через psycopg2.
- Пакетная вставка данных (executemany) для повышения производительности.
```python
    def insert_records(self, records):
        # Пакетная вставка в PostgreSQL
        with self.conn.cursor() as cursor:
            cursor.executemany(
                """
                    INSERT INTO grades (
                        user_id, 
                        oauth_consumer_key, 
                        lis_result_sourcedid, 
                        lis_outcome_service_url, 
                        is_correct, 
                        attempt_type, 
                        created_at) 
                    VALUES (%s, %s, %s, %s, %s,%s,%s)
                """, 
                [
                    (
                        record["user_id"], 
                        record["oauth_consumer_key"], 
                        record["lis_result_sourcedid"], 
                        record["lis_outcome_service_url"], 
                        record["is_correct"], 
                        record["attempt_type"], 
                        record["created_at"],
                    )
                    for record in records
                ]
            )
            self.conn.commit()
```
### Этап 4. Логирование
Настроена библиотека logging с форматом:
[%(asctime)s]: (levelname)s: %(message)s
Создан функция для удаления файлов старше 3 дней.
```sql
class Logger:
    
    def __init__(self, log_dir="grades_logs"):
        self.log_dir = log_dir
        self.logger = None
        
    def setup_logging(self):
        # Настройка логирования и ротации
        logging.basicConfig(
            filename=f'{self.log_dir}/log_{str(datetime.now().date())}.txt',
            format='%(asctime)s: %(levelname)s: %(message)s',
            level=logging.INFO,
            encoding='utf-8',
            datefmt='%Y-%m-%d %H:%M:%S')
        self.delete_old_files()
        return logging
                
    def delete_old_files(self):
        date1 = datetime.now().date()
        date2 = date1 - timedelta(days=1)
        date3 = date2 - timedelta(days=1)
        files = list(Path(self.log_dir).glob('*'))
        files = [f for f in files if f.is_file()]
        for file in files:
            if not re.match(rf'{self.log_dir}\\log_({date1}|{date2}|{date3}).txt', str(file)):
                os.remove(file)
```

Файл log:

<img width="795" height="273" alt="image" src="https://github.com/user-attachments/assets/386558a2-9434-429a-ba5e-be773807b295" />


### Этап 5. Агрегация и выгрузка в Google Sheets

Использование gspread для записи в Google Sheets (аутентификация через credentials).
```python
    def export_to_sheets(self, metrics, service_account_file, sheet_id):
        # Выгрузка в Google Sheets
        SCOPES = [
                'https://www.googleapis.com/auth/spreadsheets',
                'https://www.googleapis.com/auth/drive'
            ]
        creds = Credentials.from_service_account_file(service_account_file, scopes=SCOPES)
        client = gspread.authorize(creds)
        spreadsheet = client.open_by_key(sheet_id)
        worksheet = spreadsheet.sheet1
        new_row = []
        today = datetime.today().strftime("%Y-%m-%d")
        new_row.append(today)
        for key in ["DAU", "attempts_total", "submits_correct", "submits_incorrect", "submits_total", "runs"]:
            new_row.append(metrics[key])
        worksheet.append_row(new_row)
```

Лист GoogleSheets:
<img width="927" height="187" alt="image" src="https://github.com/user-attachments/assets/8553f611-ef4c-40bf-9563-0a2e9c753bb1" />


### Этап 6. Отправка email‑уведомлений

Настройка SMTP‑сервера через smtplib и ssl (Gmail или корпоративный сервер).
Формирование письма с:
* темой: «Отчёт по обработке данных грейдера — [дата]»;
* телом письма: метрики, ссылка на отчет GoogleSheets

```python
class NotificationService:
    def send_email(self, sender_email, password, recipient, metrics):
        msg = EmailMessage()
        today = datetime.today()
        subject = f"Отчет по грейдеру за {today}"   
        message = self.generate_email(metrics)
        msg.set_content(message)
        msg['Subject'] = subject
        msg['From'] = sender_email
        msg['To'] = recipient
        smtp_server = "smtp.mail.ru"
        port = 465
        context = ssl.create_default_context()

        with smtplib.SMTP_SSL(smtp_server, port, context=context) as server:
            server.login(sender_email, password)
            server.send_message(msg)
```

Пример письма:
<img width="1477" height="633" alt="image" src="https://github.com/user-attachments/assets/1f6ff383-89b2-4c94-b208-b3ac17846230" />

Основной скрипт:
```python
def main():

    logger = Logger().setup_logging()
    logger.info("Скрипт запущен. Начинаем получение данных из API...")
    
    api_client = APIClient(CLIENT, CLIENT_KEY,API_URL)
    processor = DataProcessor()
    db_manager = DatabaseManager(HOST, PORT, DATABASE, USER, PASSWORD)
    report_gen = ReportGenerator()
    
    notifier = NotificationService()
    
    try:
        # Имитация запроса к API
        logger.info("Отправляем запрос к API грейдера...")
        raw_data = api_client.fetch_data(START, END)
        #print(raw_data)
        logger.info("Данные успешно получены из API")
    
        # Имитация обработки данных
        logger.info("Начинаем обработку и валидацию данных...")
        processed_data = [processor.parse(record) for record in raw_data]
        logger.info("Данные обработаны и валидированы")
    
        # Имитация загрузки в БД
        logger.info("Загружаем данные в PostgreSQL...")
        db_manager.insert_records(processed_data)
        logger.info("Данные успешно загружены в PostgreSQL")
    
        # Имитация агрегации и выгрузки в Google Sheets
        logger.info("Формируем ежедневный отчёт...")
        report_metrics = report_gen.aggregate_daily_metrics(processed_data)
        report_gen.export_to_sheets(report_metrics, SERVICE_ACCOUNT_FILE, SHEET_ID)
        logger.info("Отчёт сформирован и выгружен в Google Sheets")
        
        logger.info("Отправляем отчет по почте...")
        notifier.send_email(SENDER_EMAIL, PASSWORD_EMAIL, RECIPIENT, report_metrics)
        logger.info("Письма отправлены")
        logging.shutdown()
	except Exception as e:
        logger.error(f"Произошла ошибка: {str(e)}")
        print(e)

    finally:
        logger.info("Работа скрипта завершена")
```

## Технологический стек
- **Язык программирования**: Python 3.x.
- **Базы данных**: PostgreSQL.
- **Библиотеки Python**: requests, psycopg2, logging, gspread, smtplib, ssl.
- **API**: REST API грейдера, Google Sheets API.
- **Инструменты**: Google Sheets, SMTP‑сервер (Gmail/корпоративный).
