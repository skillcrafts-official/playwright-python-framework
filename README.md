# Шпаргалка по PLAYWRIGHT
Playwright можно использовать в синхронном и асинхронном стиле.

## База

### 1. Установка
```bash
pip install playwright
playwright install
```
Команда `playwright install` скачает браузеры (Chromium, Firefox, WebKit) в отдельную папку, не в системные. После этого всё готово.

### 2. Архитектура: Browser → Context → Page
Главная сила Playwright — разделение на три уровня:
- `Browser` — сам браузер (Chromium, Firefox, WebKit). Это тяжелый процесс, его запускаем один раз.
- `BrowserContext` — изолированная сессия, как новое окно в режиме инкогнито. У каждого контекста свои куки, localStorage, кэш. Именно сюда передаются настройки (размер окна, геолокация, язык).
- `Page` — одна вкладка внутри контекста. С ней мы и работаем: переходим по URL, кликаем, читаем текст.

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=False)      # Browser
    context = browser.new_context(viewport={'width': 1280, 'height': 720})  # Context
    page = context.new_page()                        # Page
    # ... работаем ...
    browser.close()
```
Контекст можно не создавать явно — тогда `browser.new_page()` создаст страницу в дефолтном контексте. Но для шпаргалки запомни именно такую трёхслойную структуру.

### 3. Первый полноценный скрипт
Создадим файл `test_playwright.py`:
```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)  # False – увидишь окно
    context = browser.new_context()
    page = context.new_page()

    page.goto('https://example.com')
    print(page.title())              # Заголовок страницы
    page.screenshot(path='example.png')
    
    context.close()
    browser.close()
```
Запусти — получишь скриншот и заголовок в консоли. Это базовый скелет, от которого всегда можно отталкиваться.

### 4. Локаторы — сердце Playwright
Локатор описывает, что именно надо найти на странице. Playwright рекомендует использовать встроенные методы, потому что они автоматически ждут появления элемента (auto-waiting).

**Основные способы поиска:**
```python
page.get_by_role('button', name='Войти')   # кнопка с текстом «Войти»
page.get_by_label('Email')                 # поле по связанной метке <label>
page.get_by_placeholder('Введите имя')     # по атрибуту placeholder
page.get_by_text('Добро пожаловать')       # чистый текст на странице
page.get_by_test_id('submit-button')       # по data-testid (очень надёжно)
page.locator('css=div.content')            # CSS-селектор
page.locator('xpath=//button[@type="submit"]')  # XPath (если без него никак)
```
Два важнейших правила:
- Всегда стараться использовать `get_by_role`, `get_by_label`, `get_by_test_id` — они семантичные и ломаются реже.
- `page.locator()` — универсальный fallback для CSS и XPath.

### 5. Основные действия (с автоожиданием)

Playwright перед каждым действием сам дожидается, пока элемент станет видимым, доступным для клика и т.д. Никаких `time.sleep()`.
```python
# Переход
page.goto('https://example.com/login')

# Клик
page.get_by_role('button', name='Войти').click()

# Ввод текста (очищает поле и вводит)
page.get_by_label('Логин').fill('admin')

# Ввод по одному символу (как настоящий пользователь)
page.get_by_label('Логин').type('admin', delay=100)

# Выбор из выпадающего списка
page.get_by_label('Страна').select_option('Россия')

# Чтение текста
print(page.get_by_role('heading', name='Добро пожаловать').text_content())

# Получение значения атрибута
print(page.get_by_label('Логин').get_attribute('value'))
```
**Явные ожидания** нужны редко, но бывают:
```python
page.wait_for_url('**/dashboard')          # дождаться URL
page.wait_for_load_state('networkidle')    # все сетевые запросы завершены
page.get_by_role('alert').wait_for()       # дождаться появления диалога
```

### 6. Проверки (assertions)

Playwright предлагает удобные проверки через `expect`. Они автоматически перепроверяют состояние некоторое время (как мягкое ожидание), пока не станет истиной или не истечёт таймаут.
```python
from playwright.sync_api import expect

expect(page).to_have_title("Заголовок")
expect(page).to_have_url("https://...")
expect(page).to_have_url(re.compile(r"\/dashboard$"))  # Можно regex

expect(page.get_by_role('button')).to_be_enabled()
expect(page.get_by_text('Успешно')).to_be_visible()
expect(page.locator('.error')).to_have_text('Неверный пароль')
expect(page.locator('.items')).to_have_count(5)

expect(locator).to_be_visible()
expect(locator).to_be_hidden()
expect(locator).to_be_enabled()
expect(locator).to_be_disabled()
expect(locator).to_be_checked()               # для чекбоксов
expect(locator).to_contain_text("часть текста")
expect(locator).to_have_text("точный текст")
expect(locator).to_have_value("значение поля")
expect(locator).to_have_count(5)              # количество элементов
expect(locator).to_have_attribute("href", "/home")
expect(locator).to_have_css("color", "rgb(255, 0, 0)")
expect(locator).to_be_focused()
```
Это делает тесты невероятно стабильными.

### 7. Как получить (создать) Locator
Locator создаётся вызовом методов страницы или другого локатора. Самые часто используемые методы (в порядке рекомендуемости):

#### Семантические методы (рекомендованы Playwright)
```python
page.get_by_role('button', name='Войти')
page.get_by_label('Email')
page.get_by_placeholder('Введите имя')
page.get_by_text('Добро пожаловать')        # точное совпадение текста
page.get_by_text('пожаловать', exact=False) # подстрока
page.get_by_title('Показать подсказку')     # атрибут title
page.get_by_alt_text('Логотип')             # атрибут alt у картинок
page.get_by_test_id('submit-button')        # data-testid="submit-button"
```

#### Универсальные CSS / XPath
```python
page.locator('css=div.content')
page.locator('#login-button')
page.locator('xpath=//button[@type="submit"]')
```
Примечание: `page.locator('button')` автоматически определяет, что это CSS. Можно явно указывать префикс `css=` или `xpath=`.

#### Создание от другого локатора (цепочки)
Локатор можно получить от другого локатора для сужения области поиска:
```python
form = page.locator('#login-form')
submit_button = form.locator('button[type="submit"]')
```
Такой цепочечный вызов создаёт новый локатор, который будет искать внутри элемента, найденного предыдущим локатором.

#### Фильтрация локаторов
Можно отфильтровать список найденных элементов:
```python
# Все строки таблицы
rows = page.locator('table tbody tr')
# Из них только та, которая содержит текст "Иванов"
ivanov_row = rows.filter(has_text='Иванов')
# Или по наличию внутри другого локатора
row_with_button = rows.filter(has=page.locator('button'))
```
Можно сразу указать фильтр в методе:
```python
page.get_by_role('listitem').filter(has_text='Важно')
```

#### Приоритет выбора локаторов
Playwright рекомендует приоритет:
1. `get_by_role` — самый надёжный, так как опирается на семантику доступности (скринридеры видят то же самое).
2. `get_by_label` — хорошо для полей ввода с тегом <label>.
3. `get_by_test_id` — идеален, если можешь добавить data-testid в вёрстку. Ломается только при удалении атрибута.
4. **Текстовые методы** — когда нет других зацепок.
5. **CSS/XPath** — самый хрупкий, используй только если ничего другого не осталось.

### 8. Как прочитать localStorage страницы
#### Через `page`:
```python
page.evaluate("localStorage.getItem('token')")
```
#### Через `context`:
``` python
storage = page.context.storage_state()
local_storage = storage.get('origins', [{}])[0].get('localStorage', [])
```

### 9. Запуск и параллелизм
Обычный запуск (один тест за другим):
```bash
pytest test_auth.py -v
```
Параллельный запуск (с `pytest-xdist`):
```bash
pip install pytest-xdist
pytest test_auth.py -v -n auto   # auto = по числу ядер
```
Каждый worker получит свой экземпляр браузера. Параллельность — на уровне процессов. Это самый простой способ ускорить прогон без изменения кода.

**Отладка**: упавший тест автоматически сохранит скриншот, видео и трассировку (если включить).
```bash
pytest --screenshot on --video on --tracing on
```
Файлы сохранятся в `test-results/`.

## Стратегии применения

### Стратегия 1: Синхронный pytest + pytest-playwright
#### 1. Что ставим
```bash
pip install pytest playwright pytest-playwright
playwright install chromium
```
Плагин `pytest-playwright` дарит нам магические фикстуры: `page`, `browser`, `context`
и тонны полезных CLI-опций.

#### 2. Структура проекта
```text
project/
├── conftest.py          # фикстуры и настройки
├── test_auth.py         # тесты процедуры входа
└── pytest.ini           # (опционально) конфиг pytest
```

#### 3. `conftest.py` — база
Здесь мы скажем, как запускать браузер и в каком разрешении.

##### 1. Параметры `browser.launch()`
Эти параметры отвечают за то, **как именно стартует сам процесс браузера**.
```python
browser = p.chromium.launch(
    headless=False,          # ★ Визуальный запуск: True = без окна, False = видно браузер
    slow_mo=50,              # Искусственная задержка (мс) между каждым действием (удобно для отладки)
    args=[                   # Аргументы командной строки Chromium
        '--start-maximized',
        '--disable-gpu',
        '--no-sandbox'       # Часто нужно в Docker/CI
    ],
    channel='chrome',        # Использовать установленный Chrome вместо Chromium ('chrome', 'msedge', etc.)
    executable_path='/custom/path/to/chrome',  # Путь к бинарнику, если channel не подходит
    downloads_path='./downloads',  # Папка для загружаемых файлов
    proxy={                  # Настройка прокси
        'server': 'http://proxy.example.com:8080',
        'username': 'user',
        'password': 'pass'
    },
    ignore_https_errors=True, # Игнорировать ошибки SSL-сертификатов
    firefox_user_prefs={     # Только для Firefox: предпочтения
        'browser.privatebrowsing.autostart': True
    },
    timeout=30000            # Таймаут запуска браузера (мс)
)
```
**Ключевые моменты:**
- `headless=False` — тот самый «визуальный запуск», чтобы видеть окно браузера. В CI всегда `True`.
- `slow_mo` — спасение при написании/отладке тестов, когда нужно глазами успеть проследить за происходящим.
- `args` — бесценен в контейнерах (`--no-sandbox`, `--disable-dev-shm-usage`) и для фиксации размера окна.
- `channel` — удобный способ переключиться на Chrome или Edge, установленные в системе, без указания полного пути.

##### 2. Параметры `browser.new_context()`
Контекст — это изолированная среда: свои куки, localStorage, разрешения. Параметры задают **условия работы страниц внутри контекста**.
```python
context = browser.new_context(
    viewport={'width': 1280, 'height': 720},  # Размер окна
    locale='ru-RU',              # Язык интерфейса браузера
    timezone_id='Europe/Moscow', # Часовой пояс
    geolocation={'latitude': 55.7558, 'longitude': 37.6173},  # Москва
    permissions=['geolocation'], # Разрешения (гео, уведомления, камера...)
    color_scheme='dark',         # Предпочитаемая тема: 'light', 'dark', 'no-preference'
    user_agent='MyApp/1.0 (Custom)',  # Подмена User-Agent
    extra_http_headers={'X-Custom-Header': 'value'},  # Дополнительные HTTP-заголовки
    storage_state='auth.json',   # Загрузка ранее сохранённого состояния (куки, localStorage)
    ignore_https_errors=True,
    record_video_dir='videos/',  # Папка для записи видео действий
    record_har_path='network.har',  # Сохранение HAR-архива сетевых запросов
)
```
Обрати внимание на `storage_state`:

Он позволяет один раз авторизоваться, сохранить состояние в файл, а затем во всех тестах просто передавать этот файл — никакого повторного логина!
```python
# Сохранение состояния после ручного входа
page = context.new_page()
page.goto('https://myapp.example.com/login')
page.get_by_label('Логин').fill('admin')
page.get_by_label('Пароль').fill('admin')
page.get_by_role('button', name='Войти').click()
page.wait_for_url('**/dashboard')
context.storage_state(path='auth.json')

# Повторное использование
context = browser.new_context(storage_state='auth.json')
page = context.new_page()
page.goto('https://myapp.example.com/dashboard')  # уже авторизован
```

##### 3. Как применять эти параметры в `pytest-playwright`
Плагин даёт две специальные фикстуры, через которые можно кастомизировать запуск, не создавая браузер и контекст вручную.

**Кастомизация аргументов запуска браузера**:
```python
# conftest.py
@pytest.fixture(scope='session')
def browser_type_launch_args(browser_type_launch_args):
    return {
        **browser_type_launch_args,
        'headless': False,
        'slow_mo': 100,
        'args': ['--start-maximized'],
    }
```
**Кастомизация контекста (viewport, locale и т.п.)**:
```python
# conftest.py
@pytest.fixture(scope='session')
def browser_context_args(browser_context_args):
    return {
        **browser_context_args,
        'viewport': {'width': 1920, 'height': 1080},
        'locale': 'en-US',
    }
```
Если нужен нестандартный контекст в конкретном тесте, можно просто создать его внутри теста, используя фикстуру `browser`:
```python
def test_with_custom_context(browser):
    context = browser.new_context(locale='fr-FR')
    page = context.new_page()
    ...
    context.close()
```
##### Пример чистого кода (без pytest)
```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    # --- Browser ---
    browser = p.chromium.launch(
        headless=False,
        slow_mo=50,
        args=['--start-maximized']
    )
    
    # --- BrowserContext ---
    context = browser.new_context(
        viewport={'width': 1280, 'height': 720},
        locale='ru-RU',
    )
    
    # --- Page (одна вкладка внутри контекста) ---
    page = context.new_page()
    
    # Работаем
    page.goto('https://example.com')
    print(page.title())
    
    # Закрываем
    context.close()
    browser.close()
```
##### Пример через фикстуры pytest + pytest-playwright
Внутренности pytest-playwright (сильно упрощённо)
```python
@pytest.fixture
def page(browser, context):
    page = context.new_page()
    yield page
    page.close()

@pytest.fixture
def context(browser, browser_context_args):
    context = browser.new_context(**browser_context_args)
    yield context
    context.close()

@pytest.fixture(scope='session')
def browser(browser_type_launch_args):
    browser = p.chromium.launch(**browser_type_launch_args)
    yield browser
    browser.close()
```
Фикстурнутый код (в `conftest.py`):
```python
import pytest

@pytest.fixture(scope='session')
def browser_type_launch_args(browser_type_launch_args):
    """Переопределяем аргументы запуска браузера."""
    return {
        **browser_type_launch_args,
        'headless': False,
        'slow_mo': 50,
    }

@pytest.fixture(scope='session')
def browser_context_args(browser_context_args):
    """Переопределяем параметры контекста по умолчанию."""
    return {
        **browser_context_args,
        'viewport': {'width': 1280, 'height': 720},
        'locale': 'ru-RU',
    }
```

Плагин сам создаст браузер (один на сессию) и для каждого теста — изолированный контекст + страницу. Никаких ручных `sync_playwright()` — просто прописываем фикстуру `page` в тесте.

