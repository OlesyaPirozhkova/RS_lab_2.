# RS_lab_2.
# RS_lab_1


_Лабораторная работа 02. Проектирование и реализация клиент-серверной
системы. HTTP, веб-серверы и RESTful веб-сервисы_

_Вариант 18_

> Выполнила: Пирожкова О.П.

> Группа: АБП-231

**Задание**

Отправка GET-запроса к
API карт Яндекса (анализ
ответа).
API для "Отзывы о
товарах" (сущность: id,
product_id, rating, text).
Заблокировать доступ к API
для UserAgent BadBot через Nginx.

**Цели:**

Цель работы:
изучить методы отправки и анализа HTTP-запросов с
использованием инструментов telnet и curl, освоить базовую настройку и
анализ работы HTTP-сервера nginx в качестве веб-сервера и обратного
прокси, а также изучить и применить на практике концепции архитектурного
стиля REST для создания веб-сервисов (API) на языке Python.

**Оборудование и программное обеспечение:**

- Операционная система: Ubuntu 20.04.6 LTS
(в рамках предоставленного образа).
- Сетевые утилиты: telnet, curl.
- Веб-сервер: nginx.
- Среда разработки:
o Интерпретатор Python 3.8+.
o Система управления пакетами python3-pip.
o Инструмент для создания виртуальных окружений python3-
venv.
o Микрофреймворк Flask для реализации REST API.
- Доступ к сети Интернет.

Для создания каталога проекта мы будем использовать встроенный
терминал. Откройте его, выбрав в верхнем меню Terminal -> New Terminal
• В открывшемся терминале выполните следующие команды для создания и
перехода в директорию проекта:

<img width="614" height="116" alt="image" src="https://github.com/user-attachments/assets/8f10bfe4-8646-432d-8ace-602cc37b447d" />
# Шаг 1. Отправка GET-запроса к API карт Яндексаю

**С использованием telnet:**

```
telnet geocode-maps.yandex.ru 80
```

**После подключения введем:**

```
GET /1.x/?apikey=ваш_ключ&geocode=Москва,+Тверская+улица,+дом+7&format=json HTTP/1.1
Host: geocode-maps.yandex.ru
```

**Получим:**

<img width="1172" height="491" alt="image" src="https://github.com/user-attachments/assets/d02094a9-3ada-4198-9088-7e7e45df6f01" />

-Код состояния: 301 Moved Permanently - постоянное перенаправление
-Протокол: HTTP/1.1
-Сервер: nginx (популярный веб-сервер)
-Заголовок Location: https://nominatim.openstreetmap.org/... - указывает на HTTPS-версию
-Тело ответа: HTML-страница с сообщением о перенаправлении

В ходе выполнения задания был отправлен GET-запрос к API OpenStreetMap через telnet. Получен ответ с кодом состояния 301 Moved Permanently, что свидетельствует о постоянном перенаправлении с HTTP на HTTPS-версию сервиса. Заголовок Location содержал URL для HTTPS-соединения. Данный ответ демонстрирует практику обязательного использования защищенных соединений в современных веб-сервисах и механизм автоматического перенаправления пользователей на безопасную версию сайта.

# Шаг 2. Определение сервиса в .proto файле

**Создала файл newsfeed.proto**

```
syntax = "proto3";

package newsfeed;

service NewsFeed {
    rpc GetLatestNews(NewsRequest) returns (stream NewsResponse) {}
}

message NewsRequest {
    string category = 1;
}

message NewsResponse {
    string title = 1;
    string content = 2;
    string category = 3;
}
```

![Скриншот 23-09-2025 210143](https://github.com/user-attachments/assets/940d3390-581b-4ef3-b982-e7d6d9da7ea2)

# Шаг 3. Генерация кода

**Выполнила в терминале команду для генерации Python-классов из .proto файла:**

```
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. newsfeed.proto
```

![Скриншот 23-09-2025 210330](https://github.com/user-attachments/assets/2e6d1733-548f-4a16-99e1-f9e272f3d93a)

# Шаг 4. Реализация сервера

**Создала файлы server.py, client.py**

![Скриншот 23-09-2025 211221](https://github.com/user-attachments/assets/c277a08f-5046-487f-9375-cf9c9da6b86f)

![Скриншот 23-09-2025 211240](https://github.com/user-attachments/assets/f90fd3ed-70cd-47a0-81ec-3e93c55acbc1)

**Написала код сервера**

![Скриншот 23-09-2025 211539](https://github.com/user-attachments/assets/110c8297-c3a7-4433-83ab-f4c0eef1f183)

```
import grpc
from concurrent import futures
import time
import newsfeed_pb2
import newsfeed_pb2_grpc

class NewsFeedServicer(newsfeed_pb2_grpc.NewsFeedServicer):
    def GetLatestNews(self, request, context):
        print(f"Получен запрос для категории: {request.category}")
        
        news_data = {
            "technology": [
                {"title": "Новый ИИ от OpenAI", "content": "Представлена новая модель искусственного интеллекта"},
                {"title": "Запуск смартфона", "content": "Samsung представил новый флагман"},
            ],
            "sports": [
                {"title": "Футбольный матч", "content": "Результаты чемпионата"},
            ]
        }
        
        news_list = news_data.get(request.category.lower(), [])
        print(f"Найдено новостей: {len(news_list)}")
        
        for i, news in enumerate(news_list):
            response = newsfeed_pb2.NewsResponse(
                title=news["title"],
                content=news["content"],
                category=request.category
            )
            print(f"Отправляю новость: {news['title']}")
            yield response
            time.sleep(2)

def serve():
    print("Запуск сервера...")
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    newsfeed_pb2_grpc.add_NewsFeedServicer_to_server(NewsFeedServicer(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    print("Сервер запущен на порту 50051")
    print("Ожидаю подключений...")
    server.wait_for_termination()

if name == 'main':
    serve()
```

**Написала код клиента**

![Скриншот 23-09-2025 211611](https://github.com/user-attachments/assets/a2c529a9-a3c3-4c22-9dd3-d3331d919ec1)

```
import grpc
from concurrent import futures
import time
import newsfeed_pb2
import newsfeed_pb2_grpc

class NewsFeedServicer(newsfeed_pb2_grpc.NewsFeedServicer):
    def GetLatestNews(self, request, context):
        print(f"Получен запрос для категории: {request.category}")
        
        news_data = {
            "technology": [
                {"title": "Новый ИИ от OpenAI", "content": "Представлена новая модель искусственного интеллекта"},
                {"title": "Запуск смартфона", "content": "Samsung представил новый флагман"},
            ],
            "sports": [
                {"title": "Футбольный матч", "content": "Результаты чемпионата"},
            ]
        }
        
        news_list = news_data.get(request.category.lower(), [])
        print(f"Найдено новостей: {len(news_list)}")
        
        for i, news in enumerate(news_list):
            response = newsfeed_pb2.NewsResponse(
                title=news["title"],
                content=news["content"],
                category=request.category
            )
            print(f"Отправляю новость: {news['title']}")
            yield response
            time.sleep(2)

def serve():
    print("Запуск сервера...")
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    newsfeed_pb2_grpc.add_NewsFeedServicer_to_server(NewsFeedServicer(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    print("Сервер запущен на порту 50051")
    print("Ожидаю подключений...")
    server.wait_for_termination()

if name == 'main':
    serve()
```

# Шаг 5. Запуск и проверка

**Запустила и проверила файлы**

![Скриншот 23-09-2025 221810](https://github.com/user-attachments/assets/494713a9-d390-41fe-8466-900ee8052e2f)

![Скриншот 23-09-2025 221822](https://github.com/user-attachments/assets/8c62e228-c443-4b0f-9c3d-7260fb3f8c14)

# Вывод 
В ходе работы освоены основы gRPC: создание .proto-контракта, кодогенерация, реализация сервера и клиента. Успешно протестировано простое взаимодействие между ними, что подтвердило усвоение принципов RPC.

