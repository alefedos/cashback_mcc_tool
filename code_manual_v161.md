# Техническое руководство для программистов — `cashback_mcc_tool_v161.html`

> Версия документа: v161.
> Источник истины: реальный код `cashback_mcc_tool_v161.html`.
> Формат проекта: **один автономный HTML-файл** без сборки и бэкенда.
> Язык UI и комментариев — русский; имена функций/переменных/ключей `localStorage` — латиницей, как в коде.

## Содержание
1. [Общая архитектура](#1-общая-архитектура)
2. [Структура файла](#2-структура-файла)
3. [Модель данных](#3-модель-данных)
4. [Глобальное состояние](#4-глобальное-состояние)
5. [Ключи localStorage](#5-ключи-localstorage)
6. [Парсинг и нормализация MCC](#6-парсинг-и-нормализация-mcc)
7. [Парсер пользовательских категорий](#7-парсер-пользовательских-категорий)
8. [Движок расчёта кэшбэка](#8-движок-расчёта-кэшбэка)
9. [Импорт / экспорт JSON](#9-импорт--экспорт-json)
10. [Вкладки и рендереры](#10-вкладки-и-рендереры)
11. [Генерация документа](#11-генерация-документа)
12. [Сравнение MCC](#12-сравнение-mcc)
13. [Снимок месяца](#13-снимок-месяца)
14. [Вспомогательные функции](#14-вспомогательные-функции)
15. [Полный справочник функций](#15-полный-справочник-функций)
16. [Изменения v160 (слияние вкладок)](#16-изменения-v160-слияние-вкладок)
17. [Изменения v161 (активна всегда, UI карточки, слайдер ширины)](#17-изменения-v161-активна-всегда-ui-карточки-слайдер-ширины)
18. [Известный техдолг и оговорки](#18-известный-техдолг-и-оговорки)

---

## 1. Общая архитектура

### 1.1. Базовые принципы
- Всё приложение находится в одном HTML-файле.
- Хранилище данных — `localStorage`.
- Резервные копии и перенос состояния — через JSON.
- UI построен как простая SPA на ванильном JavaScript.
- Состояние хранится в наборе глобальных переменных.
- Большая часть UI перерисовывается целиком функциями `render*`.

### 1.2. Внешние зависимости
В `<head>` используются внешние ресурсы:
- Tailwind CDN;
- FontAwesome;
- веб-шрифты.

Логика приложения от них не зависит, но внешний вид в офлайне может упроститься.

### 1.3. Точка входа
```js
window.onload = initApp;
```

### 1.4. Отладочный мост
В конце файла экспортируется:
```js
window.CashbackTool = { currentData, parseMCCs, computeEffectiveForProduct };
```

---

## 2. Структура файла

Условно файл состоит из четырёх больших частей:
1. `<head>` со стилями и внешними подключениями.
2. `<body>` со статической разметкой вкладок и модалов.
3. Основной inline-скрипт со всей логикой приложения.
4. Хвост из внешних/инъецированных скриптов окружения, не относящихся к бизнес-логике.

### Основные DOM-узлы

#### Вкладки
- `#tab-editor`
- `#tab-document` (v160 — заменил `tab-selector`)
- `#tab-monthly`
- `#tab-compare`
- `#monthly-tab-label`

#### Представления
- `#view-editor` (v160 — объединённое с функциями бывшего `view-selector`)
- `#view-results`
- `#view-monthly`
- `#view-compare`

#### Контейнеры
- `#products-editor` (v160 — единственный контейнер карточек)
- `#result-html`
- `#result-plain`
- `#monthly-result-html`
- `#monthly-result-plain`
- `#compare-result-html`
- `#compare-result-plain`
- `#compare-table-container`

#### Поля селектора (v160 — перенесены внутрь `view-editor`)
- `#month-input`
- `#user-mcc-input`
- `#user-custom-categories`
- `input[name="mcc-scope"]`

#### Модалы
- `#category-modal`
- `#details-modal`
- `#import-options-modal`

---

## 3. Модель данных

### 3.1. Корневой объект
```js
currentData = {
  products: Product[],
  lastUpdated: string | null
}
```

### 3.2. Product
```js
{
  id: string,
  displayName: string,
  bank: string,
  fullName: string,
  styling: {
    textColor: string,
    bgColor: string,
    bold: boolean
  },
  productNotes: string,
  exceptions: string,
  categories: Category[],
  activationMonthlySum: string,
  minPurchase: string,
  maxPurchase: string,
  limits: {
    categories: string,
    total: string
  },
  links: string,
  cashbackForm: "rubles" | "points",
  accrualTiming: "immediately" | "after_processing" | "monthly" | "other",
  monthlyAccrual: {
    period: string,
    paymentDate: string
  },
  accrualTimingOther: string,
  pointsDetails: {
    balance: number,
    expirationType: "full" | "fifo",
    usage: string,
    expirationDates: string,
    conversion: string
  }
}
```

### 3.3. Category
```js
{
  id: string,
  internalName: string,
  cashbackPercent: number,
  mccInput: string,
  notes: string,
  restrictions: string,
  isRest: boolean,
  isAdditive: boolean,
  alwaysActive: boolean,
  ecoYandex: "" | "yes" | "no",
  ecoSber: "" | "yes" | "no"
}
```

### 3.4. Смысл ключевых полей
- `restrictions` — короткая пометка рядом с продуктом в документе.
- `notes` — более подробное описание категории.
- `alwaysActive` (v161) — категория кэшбэка действует независимо от выбора (всегда выбрана, не сбрасывается кнопкой «Снять все»).
- `ecoYandex` / `ecoSber` — только визуальные справочные флаги; расчёт не меняют.
- `isRest` — базовая категория «Все остальные» для конкретного продукта.
- `isAdditive` — суммируемая категория, добавляющая процент к основной повышенной.

---

## 4. Глобальное состояние

| Переменная | Назначение |
|---|---|
| `currentData` | Текущая база продуктов |
| `currentView` | Активное представление |
| `editingProductId` | Продукт в inline-редактировании |
| `editingCategoryId` | Редактируемая категория в модале |
| `currentProductForCategory` | Продукт, к которому относится модал категории |
| `currentSelections` | Выбранные категории по продуктам |
| `categoryPercentOverrides` | Ручные проценты на месяц |
| `userCustomCategories` | Текст поля «Ваши категории» |
| `currentMonthlySnapshot` | Сохранённый месячный документ |
| `currentImportData` | Распарсенный импортируемый JSON |
| `currentImportActions` | Список доступных действий импорта |
| `currentCompareViewMode` | Режим вкладки сравнения: `list` / `table` |
| `window.lastGenerated` | Последний сформированный документ |
| `window.lastCompare` | Последние результаты сравнения MCC |
| `DEFAULT_MCC_LIST` | Набор MCC по умолчанию |

---

## 5. Ключи localStorage

| Ключ | Содержимое |
|---|---|
| `cashback_mcc_data` | Основная база (`currentData`) |
| `cashback_mcc_selections` | Выбранные категории |
| `cashback_mcc_overrides` | Ручные проценты |
| `cashback_mcc_custom_categories` | Текст поля «Ваши категории» |
| `cashback_mcc_monthly_snapshot` | Снимок документа для вкладки месяца |
| `cashback_mcc_default_data` | Сохранённая база-умолчание |
| `cashback_mcc_default_selections` | Сохранённые выборы-умолчания |
| `cashback_mcc_default_overrides` | Сохранённые override-проценты |
| `cashback_mcc_default_selector` | Сохранённое состояние селектора (месяц, MCC) |
| `cashback_mcc_welcome_shown` | Флаг показа приветственного toast |
| `cashback_mcc_card_min_width` | (v161) Сохранённое значение минимальной ширины карточки (в %) |

Все обращения к `localStorage` обёрнуты в `try/catch`.

---

## 6. Парсинг и нормализация MCC

### `parseMCCs(input)`
Базовый парсер MCC.

Поддерживает:
- точные коды: `5411`;
- диапазоны: `5411-5499`;
- разные разделители;
- разные типы тире.

Возвращает массив токенов двух типов:
- `{type:'exact', value}`
- `{type:'range', start, end}`

### `normalizeMCCInput(rawInput)`
Нормализует список MCC для хранения в базе:
- убирает дубли;
- удаляет точные коды, покрытые диапазонами;
- сворачивает 3+ подряд идущих кода в диапазон;
- сортирует.

### `getMCCCoverageSet(input)`
Возвращает `Set<number>` всех кодов, покрытых вводом.

### `normalizeMCCInputWithReport(rawInput)`
Возвращает расширенный отчёт:
```js
{
  normalized,
  removed,
  beforeCount,
  afterCount,
  beforeList,
  afterList,
  originalInput
}
```
Используется для подтверждения при сохранении категории и исключений.

### `parseUserStatisticsMCCs(input)`
Специальный парсер для поля статистики пользователя:
- разворачивает диапазоны;
- **не схлопывает** последовательности обратно в диапазоны.

### `getExactMCCsFromInput(input)`
Возвращает только точные MCC из ввода.
Используется при scope-режимах генерации документа.

### `mccMatches(mcc, parsedList)`
Проверяет, совпадает ли один MCC со списком токенов.

### `mccMatchesInList(targetMCCs, parsedList)`
Проверяет, совпадает ли хотя бы один MCC из массива.

---

## 7. Парсер пользовательских категорий

### `parseUserCustomCategories(text)`
Разбирает поле «Ваши категории».

Формат строки:
```text
Название: 5411 (Пятёрочка, Перекрёсток), 5499 (Магнит)
```

Возвращаемая структура:
```js
[
  {
    name: string,
    mccCodes: number[],
    examples: {
      [mccNumber]: string[]
    }
  }
]
```

### Где используется результат
- `generateDocument()` — для замены стандартного имени MCC на пользовательское и показа примеров;
- `compareCategories()` — для режима сравнения;
- `attachMccClickHandlersForCustomNotes()` — для модального показа пользовательских примечаний по клику на MCC.

---

## 8. Движок расчёта кэшбэка

### `computeEffectiveForProduct(product, mcc, selectedCatIds, selectedSettings = {})`
Главная формула расчёта.

Возвращает:
```js
{
  percent: number,
  usedCategories: Category[],
  note: string
}
```

### Алгоритм
1. Из выбранных категорий продукта собираются категории, активные в этом месяце.
2. Среди них ищутся:
   - обычные повышенные категории, куда попадает MCC;
   - дополнительные (`isAdditive`) категории;
   - REST-категория.
3. Если есть повышенные категории:
   - берётся максимум среди обычных;
   - прибавляется сумма всех additive.
4. Если повышенных нет, но выбрана REST:
   - при отсутствии исключения берётся REST-процент;
   - при исключении результат = 0.
5. Процент округляется до десятых.

### Ключевая особенность
`selectedSettings` содержит **фактический процент на текущий месяц**, включая ручной override из селектора. Поэтому расчёт идёт не только по базовому `cashbackPercent`, но и по временно изменённому месячному значению.

---

## 9. Импорт / экспорт JSON

### `exportJSON()`
Экспортирует объект вида:
```js
{
  products,
  lastUpdated,
  selections,
  overrides,
  customCategories,
  lastMonth
}
```

### `importJSON(event)`
- читает файл через `FileReader`;
- проверяет формат;
- сохраняет объект в `currentImportData`;
- пытается показать модал с анализом различий;
- при проблемах откатывается на устойчивый импорт.

### `doSimpleResilientImport(imported)`
Полная замена базы с миграцией и защитой по продуктам.

### `analyzeAndShowImportOptions(imported)`
Строит список различий между текущей базой и импортом.

### `applyImportedSelectorState(imported, mode)`
Восстанавливает селекторное состояние из импортированного JSON.

### `applyFullReplaceImport()`
Полностью заменяет базу, выборы, override-проценты, пользовательские категории и месяц.

### `resetToSampleData()`
Сбрасывает базу либо к пользовательским умолчаниям, либо к встроенному sample-набору.

### `saveCurrentAsResetDefaults()`
Сохраняет текущее состояние как новое «умолчание для сброса».

---

## 10. Вкладки и рендереры

### `renderEditor()` (v160 — объединённый рендерер)
Рисует карточки продуктов в `#products-editor`. Каждая карточка теперь содержит:
- бейдж продукта;
- банк и полное имя;
- краткие лимиты;
- форму и сроки кэшбэка;
- **чекбокс** для каждой категории (v160);
- **поле процента** для ручного override (v160);
- **иконку вопроса** с tooltip для примечаний категории (v160);
- кнопку **Снять все** в заголовке списка категорий (v160);
- кнопки редактирования/удаления категории;
- ссылки;
- кнопку добавления категории.

### `editProductInline(productId)`
Переводит карточку в inline-форму редактирования.

### `renderEditCategoriesList(productId)`
Рендерит список категорий внутри inline-редактирования продукта.

### `saveEditedProduct()`
Считывает все поля формы продукта, нормализует исключения и сохраняет изменения.

### `addProduct()` / `deleteProduct(productId)` / `moveProduct(productId, direction)`
CRUD и сортировка продуктов.

### `enableCardDragAndDrop()`
Включает HTML5 drag&drop перестановку карточек редактора.

### `addCategoryToProduct(productId)` / `editCategory(productId, categoryId)`
Открывают модал категории.

### `saveCategoryFromModal()`
Проверяет и сохраняет категорию.

### `deleteCategory(productId, categoryId)`
Удаляет категорию.

### `renderSelector()` (v160 — DEPRECATED)
Заменён на `renderEditor()`. Функция оставлена для обратной совместимости и вызывает `renderEditor()`.

### `switchView(view)` (v160 — обновлён)
Переключает представления: `editor`, `results`, `monthly`, `compare`.
Вкладка `selector` удалена; добавлена вкладка `document` для `results`.

---

## 11. Генерация документа

### `generateDocument()` (v160 — обновлён)
Основной пайплайн:
1. Читает месяц, MCC пользователя и scope.
2. Парсит MCC статистики через `parseUserStatisticsMCCs()`.
3. Собирает `selectedPerProduct` из DOM **редактора** (v160 — раньше из селектора).
4. Выделяет категории без явных MCC в `noMCCSelected`.
5. Собирает итоговый список MCC в зависимости от scope.
6. Парсит пользовательские категории.
7. Для каждого MCC и продукта вызывает `computeEffectiveForProduct()`.
8. Сортирует продукты по убыванию процента.
9. Передаёт данные в `renderResults()`.
10. Переключает интерфейс во вкладку **Документ** (v160).

### `renderResults(month, results, selectedPerProduct, noMCCSelected, mccToCustomName, mccToCustomNote)`
Строит HTML- и plain-text версии документа.

### `downloadFile(format, forMonthly = false)`
Экспортирует сформированный документ в `txt`, `md`, `html`.

---

## 12. Сравнение MCC

### `compareCategories()`
Строит сравнение по MCC из пользовательской статистики. Переключает на вкладку **Сравнение MCC**.

### `renderCompareResults(userMCCs, perMCC, mccToCustomName, mccToCustomNote)`
Рисует список сравнения во вкладке `compare`.

### `toggleCompareView(mode)`
Переключает между `list` и `table`.

### `renderCompareTableInline()`
Строит таблицу сравнения внутри приложения.

---

## 13. Снимок месяца

### `applyAsMonthlyDocument()`
Сохраняет текущий `window.lastGenerated` в `currentMonthlySnapshot` и в `localStorage`.

### `renderMonthlyView()`
Восстанавливает и показывает сохранённый месячный документ.

### `updateMonthlyTabLabel()`
Обновляет подпись вкладки `Кэшбэк на <месяц>` по значению `#month-input`.

---

## 14. Вспомогательные функции

| Функция | Назначение |
|---|---|
| `generateId(prefix)` | Генерация id |
| `getRestCategory(product)` | Получение REST-категории продукта |
| `sortCategories(product)` | Сортировка категорий, REST последней |
| `categoryHasNoMCC(cat)` | Проверка категории без явных MCC |
| `formatLimitValue(val)` | `15000 -> 15К` |
| `formatLimits(cat, total)` | Формирование краткой строки лимитов |
| `getCashbackFormLabel(product)` | Краткая подпись формы кэшбэка |
| `getAccrualTimingLabel(product)` | Краткая подпись срока начисления |
| `getFormTimingShort(product)` | Компактная форма+срок |
| `getMCCCategoryName(mcc)` | Справочное имя MCC |
| `getEcoBadgeHTML(ecosystem, value)` | HTML бейджа Я/С |
| `getCategoryEcoBadgesHTML(cat)` | Набор бейджей для категории |
| `showProductDetails(productId, mcc)` | Модал деталей продукта |
| `closeDetailsModal()` | Закрытие модала деталей |
| `showToast(message, type)` | Всплывающее уведомление |
| `attachMccClickHandlersForCustomNotes(container)` | Клики по MCC для показа пользовательских примечаний |
| `addSampleMCCs()` / `clearUserMCCs()` | Сервисные действия над полем MCC |
| `initTailwind()` | Установка CSS-переменной `--accent` |
| `initApp()` | Инициализация приложения |

---

## 15. Полный справочник функций

### Хранилище и состояние
`initTailwind`, `loadSelections`, `saveSelections`, `updateProductSelection`, `uncheckAllForProduct`, `getDefaultMCCString`, `setDefaultMCCs`, `loadCustomCategories`, `saveCustomCategories`, `loadCategoryOverrides`, `saveCategoryOverrides`, `updateCategoryPercent`, `getEffectivePercent`, `saveToLocalStorage`, `loadFromLocalStorage`

### Миграции и sample-данные
`migrateProduct`, `migrateData`, `getRestCategory`, `getSampleData`

### MCC и парсеры
`generateId`, `parseMCCs`, `normalizeMCCInput`, `getMCCCoverageSet`, `normalizeMCCInputWithReport`, `getExactMCCsFromInput`, `mccMatches`, `parseUserStatisticsMCCs`, `mccMatchesInList`, `parseUserCustomCategories`, `sortCategories`, `categoryHasNoMCC`

### Подписи и форматирование
`getCashbackFormLabel`, `getAccrualTimingLabel`, `getFormTimingShort`, `getMCCCategoryName`, `formatLimitValue`, `formatLimits`, `getEcoBadgeHTML`, `getCategoryEcoBadgesHTML`

### Импорт / экспорт / сброс
`exportJSON`, `applyImportedSelectorState`, `importJSON`, `doSimpleResilientImport`, `analyzeAndShowImportOptions`, `closeImportOptionsModal`, `applyAllImportOptions`, `applyFullReplaceImport`, `applySelectedImportOptions`, `resetToSampleData`, `saveCurrentAsResetDefaults`, `clearAllProducts`

### Редактор (v160 — объединённый)
`addProduct`, `moveProduct`, `enableCardDragAndDrop`, `renderEditor`, `editProductInline`, `renderEditCategoriesList`, `cancelEditProduct`, `saveEditedProduct`, `deleteProduct`, `addCategoryToProduct`, `editCategory`, `saveCategoryFromModal`, `closeCategoryModal`, `deleteCategory`

### Документ / сравнение
`renderSelector` (deprecated), `addSampleMCCs`, `clearUserMCCs`, `generateDocument`, `compareCategories`, `renderCompareResults`, `toggleCompareView`, `renderCompareTableInline`, `downloadComparePlain`, `downloadCompareTable`, `computeEffectiveForProduct`, `renderResults`

### Модалы / копирование / view-state
`showProductDetails`, `closeDetailsModal`, `copyAsText`, `copyAsMarkdown`, `copyPlainTextArea`, `downloadFile`, `switchView`, `showToast`, `initApp`

### Вкладка месяца
`updateMonthlyTabLabel`, `applyAsMonthlyDocument`, `renderMonthlyView`, `copyAsTextForMonthly`, `copyAsMarkdownForMonthly`, `copyPlainTextAreaForMonthly`, `attachMccClickHandlersForCustomNotes`

---

## 16. Изменения v160 (слияние вкладок)

### 16.1. Архитектура: единый интерфейс
- Вкладки **«Редактор базы»** и **«Выбор на месяц»** объединены в один интерфейс на базе `view-editor`.
- Вкладка **«Выбор на месяц»** (`tab-selector` / `view-selector`) полностью удалена из HTML и навигации.
- Добавлена новая вкладка **«Документ»** (`tab-document`), которая отображается при переключении на `view-results`.

### 16.2. Карточка продукта: чекбоксы и проценты
Функция `renderEditor()` переработа:
- **Чекбокс** (`product-cat-checkbox`) слева от названия каждой категории.
- **Поле процента** (`category-percent-input`) справа от названия категории для ручного override на текущий месяц.
- **Иконка вопроса** (`fa-circle-question`) с CSS-tooltip для примечаний категории (`cat.notes`).
- **Кнопка «Снять все»** (`uncheckAllForProduct`) в заголовке блока категорий, выровнена вправо.

### 16.3. Tooltip для примечаний категории (новый CSS)
Добавлены стили `.cat-note-tooltip` и `.tooltip-text`:
- Появление при наведении с задержкой **200 мс** (`transition-delay: 0.2s`).
- Исчезновение через **1 с** после ухода курсора (`visibility 0s 1s`).
- Дублирование функции при клике/тапе через класс `.active` (переключается через `this.classList.toggle('active')`).
- Тёмный фон `#1e2937`, белый текст, максимальная ширина 250px.

### 16.4. Верхние контролы селектора
Поля месяца, MCC-кодов, пользовательских категорий и радио-кнопки scope (`mcc-scope`) перенесены в верхнюю часть `view-editor` в белый блок `bg-white border rounded-3xl`.

### 16.5. Кнопки действий
- **«Сформировать документ»** и **«Сравнить категории»** добавлены к кнопкам редактора.
- Все пять кнопок присутствуют и **вверху** страницы (под заголовком), и **внизу** (после сетки карточек).
- Кнопка **«Перейти к выбору категорий на месяц →»** удалена.

### 16.6. Генерация документа
- `generateDocument()` теперь читает чекбоксы и поля процента из `#products-editor` вместо `#products-selector`.
- После генерации переключает на вкладку **«Документ»** (через `switchView('results')`, который активирует `tab-document`).

### 16.7. Навигация «назад»
Все кнопки возврата из `view-results`, `view-compare` и `view-monthly` теперь ведут в `view-editor` вместо удалённого `view-selector`.

### 16.8. Инициализация
- `initApp()` инициализирует поля `user-mcc-input` (значение по умолчанию) и `user-custom-categories` (обработчик `input` + сохранение), которые раньше инициализировались в `renderSelector()`.
- `renderSelector()` объявлен deprecated и перенаправляет на `renderEditor()`.

### 16.9. Затронутые функции (полный список)

| Функция | Изменение |
|---|---|
| `renderEditor()` | Полная переработка: чекбоксы, проценты, tooltips, «Снять все», event listeners |
| `generateDocument()` | Контейнер изменён с `#products-selector` на `#products-editor` |
| `switchView()` | Удалена ветка `selector`, добавлена активация `tab-document` для `results` |
| `initApp()` | Добавлена инициализация полей селектора |
| `renderSelector()` | Deprecated, вызывает `renderEditor()` |

### 16.10. Обратная совместимость
- Все ключи `localStorage` оставлены без изменений.
- Поля модели данных не переименованы.
- `migrateProduct()` не требует изменений.
- Функция `renderSelector()` оставлена как заглушка для обратной совместимости.

---

## 17. Изменения v161 (активна всегда, UI карточки, слайдер ширины)

### 17.1. Модель данных: опция «Активна всегда»
- В модель `Category` добавлено булево поле `alwaysActive`.
- При включении в модальном окне (`#category-modal`) свойство сохраняется в `cat.alwaysActive = alwaysActive`.
- В `renderEditor()` и `renderEditCategoriesList()` категории с `alwaysActive === true` рендерятся с чекбоксом в состоянии `checked disabled`, который выглядит серым (`accent-slate-400 opacity-60`).
- Функция `uncheckAllForProduct()` переработана: теперь она снимает отметку только с тех чекбоксов, у которых нет атрибута `disabled` (`.querySelectorAll('.product-cat-checkbox:not([disabled])')`), благодаря чему категории с «Активна всегда» остаются выбранными.
- В функции `generateDocument()` исправлен старый баг с `closest('label')`: заменено на `closest('.category-row')`, что позволяет корректно считывать override-проценты из DOM.

### 17.2. Форма редактирования продукта
- В функции `editProductInline()` кнопка «Отмена» (`cancelEditProduct()`) продублирована в нижнем блоке кнопок рядом с «Сохранить изменения» и «Удалить продукт».
- Убрана опция и чекбокс «Жирный» (`#edit-bold`). В модели `styling` свойство `bold` при сохранении принудительно устанавливается в `false`.

### 17.3. UI карточки продукта и категорий
- Ширина поля `input` для размера кэшбэка (`category-percent-input`) сокращена до 3 символов (`style="width: 4.5ch;"`).
- В CSS (`<style>`) добавлено скрытие стрелочек (spinners) для полей `input[type="number"]`, чтобы они не перекрывали цифры в узком поле.

### 17.4. Настройка минимальной ширины карточки (слайдер)
- В верхней части вкладки «Редактор базы» добавлен регулятор-слайдер (`#card-min-width-slider`) с диапазоном от 80% до 200% (базовое значение 100% = 300px).
- При перемещении регулятора в реальном времени обновляется CSS-переменная `--card-min-width` на корневом элементе, а контейнер `#products-editor` использует CSS Grid:
  `grid-template-columns: repeat(auto-fill, minmax(min(var(--card-min-width, 300px), 100%), 1fr))`.
- Значение слайдера сохраняется в `localStorage` под ключом `cashback_mcc_card_min_width` и автоматически восстанавливается при загрузке приложения.

---

## 18. Известный техдолг и оговорки

1. **`downloadFile(format, forMonthly = false)` не использует `forMonthly` по назначению.**
   Функция всегда опирается на `window.lastGenerated`, даже если вызвана из месячной вкладки.

2. **`cashback_mcc_default_selector` сохраняется, но явно не восстанавливается при `resetToSampleData()`.**

3. **В файле присутствуют инъецированные внешние скрипты окружения.**

4. **Логика сильно завязана на глобальные переменные и прямую работу с DOM.**

5. **Внутри `renderResults()` остался неиспользуемый `usedNote`.**

> При любых дальнейших изменениях нельзя переименовывать поля модели и ключи `localStorage` без миграции через `migrateProduct()` и совместимые переходы импорта.
