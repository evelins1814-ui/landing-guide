# Инструкция Claude Design: лендинги для вакансий HRDeva

**Версия:** 2026-05-25 (заменяет драфт от 2026-05-22 — он описывал гипотетический протокол ДО реализации; теперь система живёт в проде и реальные правила другие).

## Что это

Лендинг — это HTML-страница, которую видит кандидат, когда мы ему присылаем ссылку. На лендинге он смотрит контент (видео руководителя, описание, условия) и заполняет короткую форму. Сабмит формы автоматически создаёт записи в карточке отклика (`Apply`) в Recruto CRM: видео, фото, файлы — на свои места, остальные поля — одной строкой в `ApplyCharacteristic`.

**Поток (целиком):**
1. Ты делаешь HTML-лендинг под вакансию (один лендинг = одна вакансия).
2. Кладёшь рядом `meta.json` с описанием формы.
3. Архивируешь в zip → передаёшь рекрутёру → он грузит через https://ats.hrdeva.ru / вкладка «Лендинги».
4. Лендинг живёт на `https://m.recruto.ru/<vacancy_id>` (публичная ссылка) и на `https://m.recruto.ru/<5-букв>` (персональная под конкретного кандидата).
5. Кандидат жмёт submit → SDK сам всё отправляет → данные попадают в его `Apply` в Recruto CRM.

Тебе **не нужно** писать backend, обработчик submit, экран благодарности, кнопки мессенджеров. Это всё делает наш landings-сервис. Твоя задача — **только разметка + контент + meta.json**.

---

## Структура zip-архива (обязательная)

```
landing.zip
├── index.html         ← единственная точка входа, ОБЯЗАТЕЛЬНО
├── meta.json          ← описание формы и кастомных событий, ОБЯЗАТЕЛЬНО
└── assets/            ← опционально: CSS, картинки, видео, шрифты
    ├── hero.jpg
    ├── style.css
    └── ...
```

Правила:
- **Только** эти три верхне-уровневых пути: `index.html`, `meta.json`, `assets/*`. Любые другие файлы → отказ при загрузке.
- Никаких `..` или абсолютных путей внутри zip.
- Ссылки на ресурсы в HTML — **относительные** (`assets/hero.jpg`), без `https://...`.
- Без отдельных HTML-страниц кроме `index.html`. Если нужно несколько секций — один файл, скролл/якоря.

---

## Обязательный `<head>`

В `<head>` обязательно ровно одна строка с SDK:

```html
<script src="https://m.recruto.ru/sdk/analytics.js" defer></script>
```

Если этого тега нет — сервер **откажется принимать архив**. Серверная валидация ищет регулярку `<script[^>]+src=["'][^"']*?/sdk/analytics\.js["']`.

Дополнительно желательно (но не обязательно):

```html
<meta name="robots" content="noindex,nofollow">
<meta name="viewport" content="width=device-width, initial-scale=1">
```

SDK сам инжектит `<meta name="recruto:vacancy_id">` и `<meta name="recruto:code">` на отдаче, тебе их трогать **не нужно**.

---

## Форма — обязательный шаблон

```html
<form data-recruto-form novalidate>
  <input name="applicant_name" type="text" required>
  <input name="contact_phone"  type="tel"  required>
  <input name="email"          type="email">
  <input name="experience"     type="text">
  <input name="video"          type="file" accept="video/*">
  <button type="submit">Отправить отклик</button>
</form>
```

Требования:
1. **Один тег `<form>` с атрибутом `data-recruto-form`.** SDK ищет именно этот атрибут — без него форма не будет работать. Можно несколько форм — SDK подцепит все (но обычно одна на лендинг).
2. **Никакого `action=`, `onsubmit=`, `method=`** — SDK сам перехватывает submit. Если оставишь свой обработчик — конфликт с SDK, форма уйдёт в никуда.
3. **Атрибут `name` у каждого input/select/textarea** — английский snake_case (`applicant_name`, `contact_phone`, `salary_expectation`). **Никакой кириллицы в `name`.** Кириллица в `placeholder`/`label` — пожалуйста.
4. **HTML-имена `name` должны полностью совпадать с тем, что в `meta.json.form.fields[].name`.** Сервер сверяет: если в meta задекларировано поле, а в HTML его нет — отказ при загрузке.
5. Кнопка submit — `<button type="submit">`.
6. `novalidate` на форме — рекомендуется, чтобы браузер не блокировал submit своими ошибками; SDK сам валидирует против meta.json.

---

## meta.json — обязательный формат

```json
{
  "title": "Recruto · Младший рекрутер удалённо",
  "description": "Короткое описание для отладки и для og-тегов в будущем.",
  "sdk_version": "1.0",
  "custom_events": ["hero_apply_cta", "watch_director_video"],
  "form": {
    "submit_label": "Отправить отклик",
    "fields": [
      {"name": "applicant_name", "type": "text",   "required": true,  "role": "name",  "label": "Имя и фамилия"},
      {"name": "contact_phone",  "type": "tel",    "required": true,  "role": "phone", "label": "Телефон"},
      {"name": "email",          "type": "email",  "required": false, "role": "email", "label": "Email"},
      {"name": "experience",     "type": "select", "required": false, "options": ["<1","1-3","3-5","5+"], "label": "Опыт, лет"},
      {"name": "video",          "type": "file",   "required": false, "accept": "video/*", "max_size_mb": 60, "label": "Видео-визитка"}
    ]
  }
}
```

### Поля верхнего уровня
- `title` (string) — заголовок страницы; используется в `<title>` thank-you экрана.
- `description` (string) — справочное.
- `sdk_version` (string) — пиши `"1.0"` пока не сообщим иное.
- `custom_events` (array of strings) — **все значения `data-track`** должны быть здесь задекларированы (см. секцию «Кастомные клики» ниже). Если в HTML есть `data-track="foo"`, а `foo` нет в `custom_events` — это не ломает сервис, но мы не сможем эти события подписать человеческим лейблом в админке.
- `form.submit_label` (string) — текст кнопки submit. Опциональный, для отображения.
- `form.fields` (array, **минимум 1 элемент**) — описание полей.

### Описание каждого поля

| Атрибут | Тип | Обязательный | Описание |
|---|---|---|---|
| `name` | string | **да** | Английский snake_case, уникален в форме, совпадает с HTML `name`. |
| `type` | string | **да** | Один из поддерживаемых типов (см. ниже). |
| `required` | boolean | нет | `true` → SDK / сервер откажут на пустом submit. |
| `role` | `"name"`/`"phone"`/`"email"` | нет | Семантическая роль для CRM-маппинга. Без неё значение попадёт в `ApplyCharacteristic`, но не в `name_value`/`phone_value`/`email_value`. **Для известных полей ставь обязательно.** |
| `label` | string | рекомендуется | Русский лейбл — попадёт в `ApplyCharacteristic` в карточке отклика. Без него используется `name` (некрасиво). |
| `options` | array | обязателен для `select` | Список значений. |
| `accept` | string | для `file` | MIME-маска: `"video/*"`, `"image/*"`, `"application/pdf,image/*"`. |
| `max_size_mb` | int | для `file` | Лимит размера файла. Дефолт 10 MB. |
| `pattern` | regex | нет | Дополнительная валидация для `text`/`tel`. |
| `max_length` | int | нет | Только для текстовых типов. |
| `min`/`max` | int | для `number`/`date` | Границы. |

### Поддерживаемые `type`

| `type` | HTML-эквивалент | Что отправляется |
|---|---|---|
| `text` | `<input type="text">` или `<textarea>` (для textarea используй `type: "textarea"`) | string |
| `textarea` | `<textarea>` | string |
| `tel` | `<input type="tel">` | string |
| `email` | `<input type="email">` | string |
| `number` | `<input type="number">` | **число** (JS Number). Если хочешь строку — ставь `type: "text"` в meta. |
| `select` | `<select>` | string из `options` |
| `checkbox` | `<input type="checkbox">` | boolean |
| `date` | `<input type="date">` | string `"YYYY-MM-DD"` |
| `file` | `<input type="file">` | file_id (SDK загружает отдельно) |

⚠️ **`<input type="range">` поддерживается, но в meta используй `type: "text"`** — SDK его читает как строку, бэк нумерик не ждёт. Если нужно числовое значение из слайдера — добавь видимый span + JS, обновляющий значение (см. пример с `updateRange` в Itagro-лендинге).

⚠️ **`<input type="radio">` с одним и тем же `name`** работает корректно: значение selected-radio попадёт в submit. В meta задекларируй один раз: `{"name": "hours", "type": "text", ...}`.

### Куда попадают данные

После успешного submit:

| `role` | Куда в `Apply` |
|---|---|
| `name` | `Apply.name_value` (быстрый матчинг + видно в карточке) |
| `phone` | `Apply.phone_value` |
| `email` | `Apply.email_value` |
| без role / любое другое поле | склейка в одну строку `ApplyCharacteristic.characteristic_text`, формат `«<label>: <value>\n...»` |

Файлы:
| MIME | Куда |
|---|---|
| `video/*` | `ApplyVideo` (с превью через ffmpeg + scp на NAS, потом видно в плеере карточки) |
| `image/*` | `ApplyPhoto` |
| остальное | `ApplyImages` |

---

## Кастомные клики через `data-track`

Любому элементу можно повесить атрибут `data-track="<имя_события>"`. SDK при клике на этот элемент (или любого его потомка) автоматически отправит событие `click_element` с `payload.name = <имя_события>`.

Используй для:
- CTA-кнопок (`data-track="hero_apply_cta"`, `data-track="footer_apply_cta"`)
- Кликов по примерному видео / референсам (`data-track="watch_director_video"`)
- Любых интересных элементов («раскрыли FAQ», «кликнули на условия», и т.д.)

**Все `data-track` значения обязательно занеси в `meta.json.custom_events`** — иначе у админки/аналитики не будет человеческих лейблов.

Пример:
```html
<a class="btn btn-primary" href="#form" data-track="hero_apply_cta">Откликнуться</a>

<div class="video-thumb" data-track="watch_director_video">
  <a href="https://...">▶ Слово директора</a>
</div>
```

И в `meta.json`:
```json
"custom_events": ["hero_apply_cta", "watch_director_video"]
```

---

## Что трекается АВТОМАТИЧЕСКИ (тебе не надо)

SDK сам собирает и шлёт:

- `landing_open` — после первого взаимодействия (скролл/мышь/тап) — фильтр от preview-ботов мессенджеров
- `scroll_depth` — на 25/50/75/90/100%
- `time_on_page_tick` — каждые 15 секунд, плюс финальный flush на закрытии вкладки
- `landing_visible` / `landing_hidden` — Page Visibility API (свернули таб)
- `landing_close` — на pagehide, с финальной длительностью и max_scroll
- `click_phone` — автоматически на `<a href="tel:...">`
- `click_email` — автоматически на `<a href="mailto:...">`
- `click_external` — автоматически на внешние ссылки
- `form_start` — первый focus в любое поле формы
- `form_field_filled` — на blur с непустым значением (для каждого поля отдельно)
- `form_field_invalid` — при ошибке валидации submit
- `form_submit` — успешная отправка
- `form_abandoned` — закрыли вкладку с непустой формой, не отправили
- `bot_click` — клик на кнопку мессенджера на thank-you экране
- `bot_started` — кандидат реально написал в боте (от самого бота)

Плюс **device-контекст** на каждом батче: device_type (mobile/tablet/desktop), os, browser, viewport, dpr, touch, lang, tz, connection_type.

---

## Восстановление контента в Apply

Один submit формы → одна `ApplyCharacteristic` с многострочным русским текстом. Пример того, что увидит рекрутёр в карточке:

```
Заполненные поля с лендинга:
Имя: Иван Петров
Телефон: +79991234567
Email: ivan@example.com
Опыт, лет: 3-5
ВКонтакте: vk.com/ivan

Прикреплено:
  · видео: visitka.mp4
```

Файлы видны в плеере / фотогалерее карточки автоматически — без дополнительных действий.

---

## Что **запрещено** делать

| Запрет | Почему |
|---|---|
| Свой обработчик submit (`onsubmit=`, `addEventListener('submit', ...)`, `form.submit()`) | Конфликт с SDK, данные не уйдут |
| Свой экран благодарности | SDK всё равно навигирует на наш `/<code>/thanks` |
| Подключать сторонние JS-фреймворки (jQuery, Tilda, Wix-обёртки, **Wix Bundler**) | Bundler сериализует HTML в base64 и пакует blob-URLs — нам приходится распаковывать руками. Делай плоский HTML. |
| `@font-face { src: url("uuid-без-расширения") }` | Tilda-style ссылки на blob-ресурсы не выживают вне исходного бандла. Используй обычные `assets/font.woff2` или ссылки на CDN Google Fonts. |
| Внешние `<script src="evil.com/x.js">` | XSS-вектор. Сервер пока не блочит, но мы будем — не закладывайся. |
| ПДн в DOM (имена, телефоны кандидатов, ID) | 152-ФЗ + ссылка может попасть в чужие чаты, плюс мы шифруем в логах |
| Кириллица в `name`-атрибутах | Несовместимо с backend-парсером форм |
| `mailto:` или Google Forms как fallback | Это не интегрировано с CRM |
| `<form>` без `data-recruto-form` | SDK его не увидит, submit улетит в never-never-land |
| Контекстные модалки с PII / личной информацией кандидата | Та же причина, что и DOM-PII |

---

## Что **обязательно** делать

1. **Один лендинг = один `<form data-recruto-form>`.** Никаких «отправить почту менеджеру» отдельно.
2. **Все поля формы** перечислены в `meta.json.form.fields` с правильными `type` и `name`.
3. **Для известных полей** проставляй `role: "name"|"phone"|"email"`. Иначе они уйдут только в текст характеристики, не помогут CRM с матчингом.
4. **CTA-кнопки** размечай `data-track="..."` (минимум hero CTA и mid CTA).
5. **Все `data-track`** перечисли в `meta.json.custom_events`.
6. **SDK-скрипт в `<head>`** ровно один.
7. **Относительные ссылки** на ресурсы (`assets/...`), не абсолютные на твой github-pages домен.
8. **Не клади** в DOM никаких ПДн.
9. **Тестируй локально** через предпросмотр в браузере (с временной заглушкой на SDK URL — например, скачай содержимое `https://m.recruto.ru/sdk/analytics.js` и подключи локально). Submit будет фейлиться (нет бэка), но вёрстку и поведение JS можно отладить.

---

## Реальные примеры в проде

Можешь подсмотреть как уже сделанные лендинги построены:

- **Recruiter** (vacancy_id=558) — https://m.recruto.ru/558  
  Адаптировано из исходника https://evelins1814-ui.github.io/Recruiter/ (см. `landings-prepared/recruiter/` в репозитории `hrdeva`). Простой плоский HTML, форма с 9 полями включая видео-файл.

- **Itagro** (vacancy_id=544) — https://m.recruto.ru/544  
  Был обёрнут Tilda-бандлером — пришлось распаковывать (см. `landings-prepared/itagro/`). Мораль: **плоский HTML без бандлеров**, иначе работа удваивается. Форма с range-слайдерами (B2B / cold experience) — функция `updateRange` копирует значение слайдера в видимый span (см. конец `index.html`).

---

## Чек-лист перед сдачей лендинга

- [ ] zip с тремя верхнеуровневыми путями: `index.html`, `meta.json`, `assets/`
- [ ] `<script src="https://m.recruto.ru/sdk/analytics.js" defer></script>` в `<head>`
- [ ] `<form data-recruto-form>` без `action`/`onsubmit`/`method`
- [ ] Все `name` английским snake_case, совпадают с `meta.json.form.fields[].name`
- [ ] Известные поля имеют `role: "name"|"phone"|"email"`
- [ ] Видео-поле имеет `accept="video/*"` и `max_size_mb`
- [ ] Все `data-track` значения занесены в `custom_events`
- [ ] Никаких ПДн в HTML
- [ ] Никаких внешних JS-фреймворков / Tilda-bundle
- [ ] Никакого своего thank-you / своего submit
- [ ] Ссылки на ассеты — относительные
- [ ] Открыл локально — форма видна, range-слайдеры обновляют свой display

---

## Сервер откажется принимать архив, если:

- Нет `index.html` или нет `meta.json`
- `meta.json` не парсится / не объект
- Нет `meta.json.form` или `form.fields` пустой
- Поле в `form.fields` без `name` или с дублирующимся `name`
- `type` поля не из поддерживаемого списка
- `select` без `options`
- `role` не из `{name, phone, email}` (если задан)
- В `index.html` нет SDK-тега
- В `index.html` нет `<form data-recruto-form>`
- В `index.html` отсутствует поле, задекларированное в `form.fields[].name`
- Архив содержит файл вне `index.html` / `meta.json` / `assets/`
- В путях архива есть `..` или абсолютные пути

Сообщения об ошибке возвращаются на русском в ответе админки.

---

## Если что-то непонятно

Спрашивай Данила (`@pheerses`). Backend, SDK и схемы валидации — в репозитории `gitlab.deepra.ru/hrdeva` (директория `apps/landings-service/` и `apps/landings-sdk/`).
