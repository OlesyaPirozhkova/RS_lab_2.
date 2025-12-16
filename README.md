# RS_lab_2.



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


# Шаг 1. Отправка GET-запроса к API карт Яндекса

**С использованием telnet:**

```
telnet geocode-maps.yandex.ru 80
```

**После подключения введем:**

```
GET /search?q=%D0%9C%D0%BE%D1%81%D0%BA%D0%B2%D0%B0%2C%20%D0%A2%D0%B2%D0%B5%D1%80%D1%81%D0%BA%D0%B0%D1%8F%20%D1%83%D0%BB%D0%B8%D1%86%D0%B0%2C%207&format=json HTTP/1.1
Host: nominatim.openstreetmap.org

```

**Получим:**

<img width="1172" height="491" alt="image" src="https://github.com/user-attachments/assets/d02094a9-3ada-4198-9088-7e7e45df6f01" />

-Код состояния: 301 Moved Permanently - постоянное перенаправление

-Протокол: HTTP/1.1

-Сервер: nginx (популярный веб-сервер)

-Заголовок Location: https://nominatim.openstreetmap.org/... - указывает на HTTPS-версию

-Тело ответа: HTML-страница с сообщением о перенаправлении

В ходе выполнения задания был отправлен GET-запрос к API OpenStreetMap через telnet. Получен ответ с кодом состояния 301 Moved Permanently, что свидетельствует о постоянном перенаправлении с HTTP на HTTPS-версию сервиса. Заголовок Location содержал URL для HTTPS-соединения. Данный ответ демонстрирует практику обязательного использования защищенных соединений в современных веб-сервисах и механизм автоматического перенаправления пользователей на безопасную версию сайта.

# Шаг 2. API для "Отзывы о товарах" 

**Скачаем Flask**

<img width="992" height="458" alt="image" src="https://github.com/user-attachments/assets/f6922d5b-7506-4b28-b9ac-68286d271740" />

Создано Python-окружение с установленным Flask.

**Создание файла app.py**

<img width="1176" height="843" alt="image" src="https://github.com/user-attachments/assets/6697624d-f584-484e-a846-a85c19a696c9" />

**Реализованные эндпоинты API**
GET /api/reviews - Получить все отзывы
```
@app.route('/api/reviews', methods=['GET'])
def get_all_reviews():
    return jsonify({
        'reviews': reviews,
        'count': len(reviews)
    })
```
Проводим тестирование

<img width="1009" height="424" alt="image" src="https://github.com/user-attachments/assets/a0e7373c-4e7e-48b0-b685-e7d87ab2db05" />

GET /api/reviews/<id> - Получить отзыв по ID
```
@app.route('/api/reviews/<int:review_id>', methods=['GET'])
def get_review(review_id):
    for review in reviews:
        if review['id'] == review_id:
            return jsonify(review)
    return jsonify({'error': 'Отзыв не найден'}), 404
```
Проводим тестирование

<img width="1014" height="232" alt="image" src="https://github.com/user-attachments/assets/67ba2898-eb5a-4c0d-af84-581f54965bcc" />

GET /api/reviews/product/<product_id> - Отзывы по товару
```
@app.route('/api/reviews/product/<int:product_id>', methods=['GET'])
def get_reviews_by_product(product_id):
    product_reviews = [r for r in reviews if r['product_id'] == product_id]
    
    if not product_reviews:
        return jsonify({
            'message': f'Отзывы для товара {product_id} не найдены',
            'reviews': [],
            'count': 0
        })
    
    avg_rating = sum(r['rating'] for r in product_reviews) / len(product_reviews)
    
    return jsonify({
        'product_id': product_id,
        'reviews': product_reviews,
        'count': len(product_reviews),
        'average_rating': round(avg_rating, 2)
    })
```
Проводим тестирование

<img width="1021" height="481" alt="image" src="https://github.com/user-attachments/assets/7402d680-5fc5-4509-b3e8-2f384eb63412" />

 POST /api/reviews - Создать новый отзыв
```
@app.route('/api/reviews', methods=['POST'])
def create_review():
    global next_review_id
    data = request.json
    
    # Валидация
    if not data:
        return jsonify({'error': 'Требуются данные в формате JSON'}), 400
    
    required_fields = ['product_id', 'rating', 'text']
    for field in required_fields:
        if field not in data:
            return jsonify({'error': f'Отсутствует обязательное поле: {field}'}), 400
    
    # Валидация рейтинга (1-5)
    if not isinstance(data['rating'], int) or data['rating'] < 1 or data['rating'] > 5:
        return jsonify({'error': 'Рейтинг должен быть целым числом от 1 до 5'}), 400
    
    # Создание отзыва
    new_review = {
        'id': next_review_id,
        'product_id': data['product_id'],
        'rating': data['rating'],
        'text': data['text'],
        'created_at': datetime.now().isoformat()
    }
    
    reviews.append(new_review)
    next_review_id += 1
    
    return jsonify(new_review), 201
```
 Проводим тестирование

 <img width="1019" height="288" alt="image" src="https://github.com/user-attachments/assets/4045559d-19d6-43aa-b785-c8195dddfdbe" />

 GET /api/reviews/stats - Статистика по отзывам
```
@app.route('/api/reviews/stats', methods=['GET'])
def get_reviews_stats():
    if not reviews:
        return jsonify({'message': 'Нет отзывов для анализа'})
    
    total_avg = sum(r['rating'] for r in reviews) / len(reviews)
    
    products_stats = {}
    for review in reviews:
        pid = review['product_id']
        if pid not in products_stats:
            products_stats[pid] = {
                'product_id': pid,
                'review_count': 0,
                'ratings': []
            }
        products_stats[pid]['review_count'] += 1
        products_stats[pid]['ratings'].append(review['rating'])
    
    for pid in products_stats:
        ratings = products_stats[pid]['ratings']
        products_stats[pid]['average_rating'] = round(sum(ratings) / len(ratings), 2)
        products_stats[pid].pop('ratings')
    
    return jsonify({
        'total_reviews': len(reviews),
        'total_products': len(products_stats),
        'overall_average_rating': round(total_avg, 2),
        'products': list(products_stats.values())
    })
```
 Проводим тестирование

 <img width="941" height="446" alt="image" src="https://github.com/user-attachments/assets/cb749389-2bcf-42a2-9cca-64a2e7199ed0" />

Успешно создано REST API для управления отзывами о товарах


# Шаг 3. Заблокировать доступ к API для UserAgent BadBot через Nginx.

**Установка Nginx**

Успешно установлен и работает

<img width="973" height="458" alt="image" src="https://github.com/user-attachments/assets/3d7f338d-f04c-4558-823b-4ac6982cfe86" />

**Создание конфигурации для блокировки BadBot**

**Создаем конфигурационный файл:**

```
sudo bash -c 'cat > /etc/nginx/sites-available/reviews_api << "EOF"
server {
    listen 80;
    server_name localhost;
    
    # Блокировка UserAgent "BadBot"
    if ($http_user_agent ~* (BadBot|badbot|Bad_Bot)) {
        return 403;
    }
    
    # Главная страница
    location / {
        root /var/www/html;
        index index.html index.htm;
        try_files $uri $uri/ =404;
    }
    
    # Проксирование Flask API
    location /api/ {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF'
```

**активируем конфигурацию**

```
# Создаем ссылку
sudo ln -sf /etc/nginx/sites-available/reviews_api /etc/nginx/sites-enabled/

# Удаляем default если есть
sudo rm -f /etc/nginx/sites-enabled/default

# Проверяем синтаксис
sudo nginx -t

# Перезагружаем
sudo systemctl reload nginx
```

<img width="766" height="109" alt="image" src="https://github.com/user-attachments/assets/3a7f9f02-f397-48a8-b279-ee21b8e42ced" />


все работает корректно

**Проверка**

Делаем обычный запрос. Работает.

```
curl -H "User-Agent: Mozilla/5.0" http://localhost/api/reviews
```

<img width="863" height="465" alt="image" src="https://github.com/user-attachments/assets/903de721-83c6-4345-ab2f-e9bcc7987d68" />

Делаем запрос на badbot. Выдает ошибку 403.

```
curl -H "User-Agent: BadBot" http://localhost/api/reviews
```

<img width="838" height="241" alt="image" src="https://github.com/user-attachments/assets/ab7fc27a-0b1c-42fe-b766-1394e7374402" />

Также проверим варианты.

<img width="804" height="217" alt="image" src="https://github.com/user-attachments/assets/28aec5f2-ddcb-4598-b486-74e51fe3f9da" />

<img width="851" height="226" alt="image" src="https://github.com/user-attachments/assets/f9e787e8-ae52-4558-932e-c3ce35b621f1" />

Доступ к API заблокирован для UserAgent BadBot через Nginx. Блокировка работает корректно для всех вариантов написания BadBot.

# Вывод 
Успешно реализовано REST API для отзывов о товарах с полным CRUD-функционалом и защищено Nginx, который корректно блокирует доступ для вредоносных ботов по UserAgent, обеспечивая безопасность API.** Все этапы работы выполнены: HTTP-анализ через telnet/curl, разработка бизнес-логики на Flask, настройка обратного прокси с фильтрацией трафика.
