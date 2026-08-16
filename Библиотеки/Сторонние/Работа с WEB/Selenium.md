[← Оглавление](../../../README.md)

# Selenium

> _Когда сайт живёт на JavaScript — запусти настоящий браузер и кликай_

---

**Selenium** — инструмент автоматизации **настоящего браузера**: он открывает Chrome/Firefox, переходит по страницам, кликает, вводит текст, читает результат. Нужен там, где [requests](../../../Библиотеки/Сторонние/Работа%20с%20WEB/requests.md) + [BeautifulSoup](../../../Библиотеки/Сторонние/Работа%20с%20WEB/BeautifulSoup.md) бессильны: страница отрисовывается **JavaScript** уже в браузере, и в исходном HTML данных ещё нет. Второе применение — автотесты веб-интерфейса.

```bash
pip install selenium
```

## Когда нужен Selenium

| Ситуация | Инструмент |
|---|---|
| статический HTML, данные сразу в разметке | [requests](../../../Библиотеки/Сторонние/Работа%20с%20WEB/requests.md) + [BeautifulSoup](../../../Библиотеки/Сторонние/Работа%20с%20WEB/BeautifulSoup.md) (быстро) |
| контент подгружается JS, нужны клики/вход | **Selenium** (медленно, но видит всё) |
| тесты UI: проверить, что кнопка работает | **Selenium** |

> Selenium **на порядок медленнее** и тяжелее requests: он поднимает реальный браузер. Берут его только когда без браузера никак.

## WebDriver — управление браузером

**WebDriver** — объект, представляющий браузер. Через него открывают страницы и ищут элементы:

```python
from selenium import webdriver
from selenium.webdriver.common.by import By

driver = webdriver.Chrome()              # запустить браузер
driver.get('https://example.com')        # открыть страницу
print(driver.title)                      # заголовок вкладки
driver.quit()                            # закрыть браузер (обязательно!)
```

## Поиск элементов

`find_element` (первый) и `find_elements` (все) со стратегией из `By`:

```python
driver.find_element(By.ID, 'username')            # по id
driver.find_element(By.CSS_SELECTOR, 'div.price')  # по CSS-селектору
driver.find_element(By.XPATH, '//button[text()="Войти"]')  # по XPath
driver.find_elements(By.TAG_NAME, 'a')            # все ссылки (список)
```

## Действия

С найденным элементом взаимодействуют как пользователь:

```python
driver.find_element(By.ID, 'username').send_keys('Alex')   # ввести текст
driver.find_element(By.ID, 'login-btn').click()            # кликнуть
element.text                                                # прочитать текст
element.get_attribute('href')                              # атрибут
```

## Ожидания — самое важное

Браузер рисует страницу не мгновенно: элемент может ещё не появиться, когда код до него дошёл. Поэтому используют **явные ожидания** — «подожди, пока элемент появится» (до таймаута):

```python
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

element = WebDriverWait(driver, 10).until(
    EC.presence_of_element_located((By.ID, 'result'))    # ждать до 10 секунд
)
```

> **Не используй `time.sleep`** для ожидания загрузки: слишком мало — упадёт, слишком много — тормозит. `WebDriverWait` ждёт ровно столько, сколько нужно, и не дольше.

## Сквозной пример: вход и чтение результата

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

driver = webdriver.Chrome()
try:
    driver.get('https://example.com/login')                     # 1. открыть
    driver.find_element(By.ID, 'username').send_keys('Alex')    # 2. ввести логин
    driver.find_element(By.ID, 'password').send_keys('secret')  #    и пароль
    driver.find_element(By.ID, 'submit').click()                # 3. кликнуть «войти»

    welcome = WebDriverWait(driver, 10).until(                  # 4. дождаться результата
        EC.presence_of_element_located((By.CLASS_NAME, 'welcome'))
    )
    print(welcome.text)
finally:
    driver.quit()                                               # 5. закрыть всегда
```

**Как это читать.** Selenium управляет браузером как человек: открыл, заполнил поля, нажал, дождался ответа. `try/finally` гарантирует закрытие браузера даже при ошибке. Для статики это был бы overkill — там хватит [requests](../../../Библиотеки/Сторонние/Работа%20с%20WEB/requests.md) + [BeautifulSoup](../../../Библиотеки/Сторонние/Работа%20с%20WEB/BeautifulSoup.md).

## Связи

- [requests](../../../Библиотеки/Сторонние/Работа%20с%20WEB/requests.md), [BeautifulSoup](../../../Библиотеки/Сторонние/Работа%20с%20WEB/BeautifulSoup.md) — быстрый путь для статических страниц (без JS);
- [Grab](../../../Библиотеки/Сторонние/Работа%20с%20WEB/Grab.md) — фреймворк скрейпинга; для JS-страниц всё равно нужен браузер;
- [Тестирование Django приложений](../../../Фреймворки/DJANGO/Тестирование%20Django%20приложений.md) — Selenium применяют и для end-to-end тестов UI.

## Источники

- [Selenium documentation](https://www.selenium.dev/documentation/)
- [Selenium with Python](https://selenium-python.readthedocs.io/)
- [Waits — explicit and implicit](https://www.selenium.dev/documentation/webdriver/waits/)
