# ab-анализ-конверсии

> Учебный проект для **A/B-анализа конверсии** на Python: sanity-checks, CR с Δ, 95% доверительные интервалы, p-value, **CUPED**, срезы и готовый Jupyter-ноутбук.

![Python](https://img.shields.io/badge/Python-3.9%2B-blue)
![Pandas](https://img.shields.io/badge/Pandas-dataframes-lightgrey)
![SciPy](https://img.shields.io/badge/SciPy-stats-green)
![Status](https://img.shields.io/badge/status-learning-brightgreen)

---

## 📦 Что внутри

- `ab_test_mock_data.csv` — **мок-датасет** с реалистичными полями (для тренировки).
- `ab_test_template.ipynb` — **Jupyter-ноутбук**: пошаговый анализ от загрузки до вывода.
- `README.md` — этот файл.

---

## 🗂️ Схема данных

Обязательные:
- `user_id` — идентификатор пользователя (единица анализа).  
- `variant` — «A»/«B» (назначенный вариант).  
- `assigned_ts` — время назначения варианта (datetime).  
- `converted` — целевое действие (0/1).

Опциональные (желательны):
- `country`, `device`, `source` — срезы/проверки баланса.  
- `prior_activity` — предэкспериментальный ковариат для **CUPED**.  
- `sessions` — прокси экспозиции.  
- `conversion_ts` — время конверсии.  
- `revenue` — выручка (для вторичного анализа).

> ⚠️ Важно: ковариаты для CUPED считаются **до** `assigned_ts` (без leakage).

---

## 🚀 Быстрый старт

```bash
# 1) Клонируй репозиторий
git clone <your-repo-url>
cd <repo>

# 2) (Опционально) виртуальное окружение и зависимости
python -m venv .venv && source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -r requirements.txt  # см. пример ниже

# 3) Sanity-check на своём CSV
python ab_sanity_check.py --csv your_ab_export.csv --out_md sanity_report.md

# 4) Открой ноутбук и иди по шагам
jupyter lab   # или jupyter notebook / VS Code / Colab
```

Если нет `requirements.txt`, минимальный набор:
```bash
pip install pandas numpy scipy statsmodels matplotlib
```

---

## 📒 Как работать в ноутбуке (`ab_test_template.ipynb`)

1. **Импорт и `CSV_PATH`** — укажи путь к своей выгрузке.  
2. **Sanity-checks** — сплит A/B, утечки (multi-assign), временной баланс (KS), баланс ковариат (χ²/Welch t).  
3. **Основной результат** — CR по A/B, абсолютная разница (B-A), 95% CI (Wald), z-тест двух долей, p-value, относительный uplift %.  
4. **CUPED** — варианс-редукция с `prior_activity` (уже подготовлено).  
5. **Сегменты** — пример `device × source` (осторожно с множественными проверками).  
6. **Выручка (опционально)** — Welch t-test, медианы, быстрые гистограммы.  
7. **Автовывод** — готовый бизнес-текст для one-pager.

---

## 🧰 CLI-скрипты

### 1) Sanity-проверка + Markdown-отчёт
```bash
python ab_sanity_check.py --csv your_ab_export.csv --out_md sanity_report.md
```
Проверяет:
- Сплит A/B и дневное распределение назначений.  
- Мульти-назначения и дубли по `user_id`.  
- KS-тест по времени назначений (`assigned_ts`).  
- Баланс ковариат: χ² (категориальные), Welch t (числовые).

### 2) Основной анализ (конверсия / выручка)
```bash
# Конверсия
python ab_analysis_template.py --csv your_ab_export.csv --metric conversion --alpha 0.05

# Выручка
python ab_analysis_template.py --csv your_ab_export.csv --metric revenue
```
Выводит:
- Таблицу A vs B: `conversions`, `visitors`, `CR`.  
- Δ (B−A), 95% CI, относительный эффект, z-статистику и p-value.  
- CUPED-оценку (если есть `prior_activity`).  
- Пример сегментов.

---

## 🧪 Методология (кратко)

- **Единица анализа:** пользователь (`user_id`).  
- **Метрика:** `converted` (0/1), CR = `sum / count`.  
- **Тест:** двухсторонний **z-тест двух долей**; CI — Wald (для прод-кейсов можно Wilson/Newcombe).  
- **CUPED:** \( y_{adj}=y-\theta(x-\bar{x}) \) — снижает дисперсию при сильном предикторе `x`.  
- **Guardrails:** заранее фиксируй критерии остановки и следи за негативом по ключевым метрикам.  
- **Сегменты:** только как доп.анализ; учитывай множественные проверки (например, BH/FDR).

---

## 🧱 Структура проекта

```
.
├── ab_test_mock_data.csv
├── ab_test_template.ipynb
├── ab_analysis_template.py
├── ab_sanity_check.py
├── README.md
└── requirements.txt       # (опционально)
```

---

## 🧩 Пример SQL (агрегация сырых событий)

```sql
WITH assign AS (
  SELECT user_id,
         MIN(assigned_ts) AS assigned_ts,
         ANY_VALUE(variant) AS variant
  FROM ab_assignments
  GROUP BY 1
),
conv AS (
  SELECT user_id,
         MIN(event_ts) AS conversion_ts,
         1 AS converted
  FROM events
  WHERE event_name = 'signup'
  GROUP BY 1
),
attrs AS (
  SELECT user_id, country, device, source, prior_activity, sessions
  FROM user_attrs_snapshot
)
SELECT a.user_id, a.assigned_ts, a.variant,
       at.country, at.device, at.source, at.prior_activity, at.sessions,
       COALESCE(c.converted, 0) AS converted,
       c.conversion_ts
FROM assign a
LEFT JOIN conv c USING (user_id)
LEFT JOIN attrs at USING (user_id);
```

---

## 📝 One-pager (шаблон вывода)

- **Цель и гипотеза** (метрика, MDE, критерии остановки).  
- **Дизайн** (рандомизация, окно, фильтры).  
- **Итог:** A vs B — CR, Δ (п.п.), 95% CI, p-value, относительный эффект.  
- **Качество данных:** сплит/утечки/баланс.  
- **Сегменты/guardrails:** коротко, без «p-хантинга».  
- **Решение и риски** + **next steps**.

---

## 🤝 Contributing

Идеи welcome: бутстрэп-CI, BH/FDR, логистическая регрессия (ANCOVA), экспорт отчёта в PDF/HTML.

---

## 📄 License

MIT 

---

### `requirements.txt` 

```
pandas>=1.5
numpy>=1.23
scipy>=1.10
statsmodels>=0.14
matplotlib>=3.7
```
