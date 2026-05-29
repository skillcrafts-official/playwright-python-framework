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

expect(page).to_have_title('Example Domain')
expect(page.get_by_role('button')).to_be_enabled()
expect(page.get_by_text('Успешно')).to_be_visible()
expect(page.locator('.error')).to_have_text('Неверный пароль')
expect(page.locator('.items')).to_have_count(5)
```
Это делает тесты невероятно стабильными.

## Стратегии применения
