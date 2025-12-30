# Анализ изменений условий ПЛ с 28.01.2025

## Описание изменений

### 1. Изменение процентов списания баллов:
- **Знаток Вкуса** (Level 1): до **25%** суммы чека (было 50%)
- **Гурман Вкуса** (Level 2): до **50%** суммы чека (было 75%) 
- **Магистр Вкуса** (Level 3): до **75%** суммы чека (было 100%)

### 2. Изменение срока годности баллов:
- **3 года** с момента начисления (было 1 год)

---

## Анализ кодовой базы

### 1. Проценты покрытия (PercentCoverage)

#### Где хранятся:
- **Таблица БД**: `M_PromotionReward`
- **Поле**: `Amount` 
- **Условие**: `RewardType = 'B'` (B = Bonus coverage)
- **Связь**: `M_Promotion_ID` → уровень ПЛ (Level1, Level2, Level3)

#### Где используются:

**1.1. Получение процента покрытия:**
```java
// BonusMechanism.java, строка 3970-3988
public static BigDecimal getRewardAmount(int p_Promotion_ID, String p_RewardType, String trxName)
```
- Метод читает из `M_PromotionReward` где `RewardType = 'B'`
- Используется кеш: `cacheRewardAmount`

**1.2. Расчет допустимого списания:**
```java
// BonusMechanism.java, строки 2498-2528
// calcBonusOnSales() - метод расчета бонусов при продаже
CoverPercent = getRewardAmount(BP_Current_PromotionPreCondition.getM_Promotion_ID(), "B", trxName);
CoverPercent = CoverPercent.divide(Env.ONEHUNDRED, 2, BigDecimal.ROUND_HALF_UP);
BigDecimal AllowAmountToCoverByBonuses = buyRelevantAmt.multiply(CoverPercent);
```

**1.3. Возврат в API:**
```java
// LoyaltyApiService.java, строки 1869, 1870, 1219, 1220
answer.setPercentCoverage(BonusMechanism.getRewardAmount(promotionPreCondition.getM_Promotion_ID(), "B", trxName));
```

**1.4. Использование в других местах:**
- `LoyaltyAPI.java` - передача в ответ API
- `ImShopImpl.java` - отображение в интерфейсе магазина
- `LoyaltyBonusWriteOffForm.java` - форма списания бонусов
- `WebstoreAPI.java` - веб-магазин

#### Файлы для изменения:
1. **БД**: Обновить записи в `M_PromotionReward` для каждого уровня:
   - Level 1 (Знаток Вкуса): `Amount = 25` (было 50)
   - Level 2 (Гурман Вкуса): `Amount = 50` (было 75)
   - Level 3 (Магистр Вкуса): `Amount = 75` (было 100)

2. **Кеш**: После изменения БД нужно очистить кеш `cacheRewardAmount` или перезапустить приложение

---

### 2. Срок годности баллов (BurnDate)

#### Где рассчитывается:
- **Представление БД**: `IP_Loyalty_Bonus_Summary` (или `ip_loyalty_bonus_summary`)
- **Функция БД**: `ip_loyalty_get_bonus_burn()` (используется в `BonusBurning.java`, строка 54)

#### Где используется:

**2.1. Получение даты сгорания:**
```java
// LoyaltyApiService.java, строки 129-147
// getBonusHistory() - получение истории бонусов
sql = "SELECT BonusAmt, BurnDate FROM IP_Loyalty_Bonus_Summary WHERE C_BPartner_ID = ? ORDER BY BurnDate ASC";
```

**2.2. Уведомления о сгорании:**
```java
// SchedulerNotifyClientsFutureBonusBurning.java
// Уведомления за 14 дней, 7 дней, 1 день до сгорания
```

**2.3. Процесс сгорания:**
```java
// BonusBurning.java
// Планировщик, который списывает просроченные бонусы
```

#### Где устанавливается BurnDate:
**НАЙДЕНО**: BurnDate рассчитывается в представлении БД `ip_loyalty_bonus_summary`:

```sql
CASE
    WHEN ppc.m_promotionprecondition_id IS NOT NULL 
        THEN COALESCE(ppc.enddate, CURRENT_DATE + '1 year'::interval)::date
    ELSE max((jl.dateacct::timestamp with time zone + pt.netdays::character varying::interval + '1 year'::interval)::date)
END AS burndate
```

**Логика расчета:**
1. Если есть `m_promotionprecondition_id` (сертификаты/специальные условия):
   - Используется `ppc.enddate` (если задан) или `CURRENT_DATE + '1 year'`
2. Для обычных бонусов:
   - `jl.dateacct + pt.netdays + '1 year'`
   - Где `pt.netdays` - срок оплаты из `C_PaymentTerm` (обычно 0)

**Что нужно изменить:**
- В обоих местах: `'1 year'::interval` → `'3 years'::interval`
- **ВАЖНО**: Нужна условная логика для бонусов, начисленных до 28.01.2025

#### Файлы для изменения:
1. **БД - Представление `ip_loyalty_bonus_summary`**: 
   - Изменить расчет BurnDate с учетом даты начисления
   - Для бонусов с `jl.dateacct < '2025-01-28'` → оставить `'1 year'`
   - Для бонусов с `jl.dateacct >= '2025-01-28'` → использовать `'3 years'`
   
2. **Функция `ip_loyalty_get_bonus_burn()`**:
   - Использует представление `ip_loyalty_bonus_summary`
   - Изменений не требует (работает с результатом представления)

---

## План доработок

### Этап 1: Анализ БД (КРИТИЧНО)

**1.1. Найти и проанализировать:**
- [x] Структуру таблицы `M_PromotionReward` - найдено (используется в `getRewardAmount()`)
- [ ] Текущие значения `Amount` для `RewardType = 'B'` по уровням - **НУЖНО УЗНАТЬ**
- [x] Структуру представления `ip_loyalty_bonus_summary` - **НАЙДЕНО** (см. SQL ниже)
- [x] Функцию `ip_loyalty_get_bonus_burn()` - **НАЙДЕНО** (использует представление)
- [x] SQL-скрипты создания этих объектов - **ПОЛУЧЕНЫ**

**1.2. Проверить:**
- [x] Есть ли хардкод процентов в коде Java - **НЕТ**, все из БД через `getRewardAmount()`
- [x] Есть ли хардкод срока годности в коде Java - **НЕТ**, все в БД представлении
- [ ] Используется ли параметр из AD_SysConfig для срока годности - **НЕТ**, хардкод `'1 year'` в представлении

### Этап 2: Изменение процентов покрытия

**2.1. Обновление БД:**
```sql
-- Пример (нужно уточнить M_Promotion_ID для каждого уровня)
UPDATE M_PromotionReward 
SET Amount = 25 
WHERE M_Promotion_ID = <Level1_Promotion_ID> 
  AND RewardType = 'B' 
  AND IsActive = 'Y';

UPDATE M_PromotionReward 
SET Amount = 50 
WHERE M_Promotion_ID = <Level2_Promotion_ID> 
  AND RewardType = 'B' 
  AND IsActive = 'Y';

UPDATE M_PromotionReward 
SET Amount = 75 
WHERE M_Promotion_ID = <Level3_Promotion_ID> 
  AND RewardType = 'B' 
  AND IsActive = 'Y';
```

**2.2. Очистка кеша:**
- [ ] Перезапуск приложения или очистка кеша `cacheRewardAmount`
- [ ] Проверка, что новые значения применяются

**2.3. Тестирование:**
- [ ] Проверить расчет списания для каждого уровня
- [ ] Проверить API `/user` и `/receipt/bonusavailable`
- [ ] Проверить отображение в интерфейсах

### Этап 3: Изменение срока годности

**3.1. Обновление БД - Представление `ip_loyalty_bonus_summary`:**

**Текущий код:**
```sql
CASE
    WHEN ppc.m_promotionprecondition_id IS NOT NULL 
        THEN COALESCE(ppc.enddate, CURRENT_DATE + '1 year'::interval)::date
    ELSE max((jl.dateacct::timestamp with time zone + pt.netdays::character varying::interval + '1 year'::interval)::date)
END AS burndate
```

**Новый код (с учетом даты начисления):**
```sql
CASE
    WHEN ppc.m_promotionprecondition_id IS NOT NULL 
        THEN COALESCE(ppc.enddate, 
            CASE 
                WHEN CURRENT_DATE < '2025-01-28'::date THEN CURRENT_DATE + '1 year'::interval
                ELSE CURRENT_DATE + '3 years'::interval
            END)::date
    ELSE max((
        jl.dateacct::timestamp with time zone + pt.netdays::character varying::interval + 
        CASE 
            WHEN jl.dateacct::date < '2025-01-28'::date THEN '1 year'::interval
            ELSE '3 years'::interval
        END
    )::date)
END AS burndate
```

**Альтернативный вариант (более простой, но менее точный):**
Если все бонусы, начисленные до 28.01.2025, уже имеют рассчитанный BurnDate, можно просто изменить формулу для новых:
```sql
CASE
    WHEN ppc.m_promotionprecondition_id IS NOT NULL 
        THEN COALESCE(ppc.enddate, CURRENT_DATE + '3 years'::interval)::date
    ELSE max((jl.dateacct::timestamp with time zone + pt.netdays::character varying::interval + 
        CASE 
            WHEN jl.dateacct::date >= '2025-01-28'::date THEN '3 years'::interval
            ELSE '1 year'::interval
        END)::date)
END AS burndate
```

**3.2. Обработка существующих бонусов:**
- [x] **НАЙДЕНО**: BurnDate рассчитывается динамически в представлении
- [ ] **КРИТИЧНО**: Представление пересчитывается при каждом запросе
- [ ] Проверить: нужно ли обновлять уже рассчитанные BurnDate в `fact_acct_summary` или других таблицах
- [ ] Если BurnDate хранится где-то статически - нужен скрипт миграции

**3.3. Тестирование:**
- [ ] Проверить расчет BurnDate для новых начислений
- [ ] Проверить процесс `BonusBurning` (не должен сжигать бонусы раньше времени)
- [ ] Проверить уведомления о сгорании

### Этап 4: Дополнительные проверки

**4.1. Подарочные сертификаты:**
- [ ] Проверить, как работают сертификаты при 75% покрытии
- [ ] В коде есть проверки на сертификаты (`isCertificate`)
- [ ] Убедиться, что сертификаты не зависят от процента покрытия

**4.2. API и интерфейсы:**
- [ ] Обновить документацию API (если есть)
- [ ] Проверить мобильное приложение
- [ ] Проверить веб-интерфейс магазина
- [ ] Проверить интерфейс 1С

**4.3. Уведомления:**
- [ ] Проверить шаблоны уведомлений о сгорании (возможно, нужно обновить текст)
- [ ] Проверить логику уведомлений (14 дней, 7 дней, 1 день)

---

## Риски и последствия

### Риск 1: Кеширование
- **Проблема**: Проценты покрытия кешируются в `cacheRewardAmount`
- **Решение**: Очистка кеша после изменения БД

### Риск 2: Существующие бонусы
- **Проблема**: Бонусы, начисленные до 28.01.2025, должны сгорать через 1 год
- **Решение**: Условная логика в расчете BurnDate или миграция данных

### Риск 3: Обратная совместимость
- **Проблема**: API может возвращать старые значения
- **Решение**: Тестирование всех точек входа

### Риск 4: Подарочные сертификаты
- **Проблема**: Вопрос в задаче: "при 75% покрытии как будут работать подарочные сертификаты?"
- **Решение**: Анализ кода показал, что сертификаты обрабатываются отдельно, но нужно проверить

---

## Вопросы для уточнения

1. **Уровни ПЛ**: Подтвердить названия уровней и их M_Promotion_ID:
   - Level 1 = Знаток Вкуса
   - Level 2 = Гурман Вкуса (или другое название?)
   - Level 3 = Магистр Вкуса

2. **Дата применения**: Изменения с 28.01.2025 - это:
   - Для всех операций с этой даты?
   - Для всех начислений с этой даты?

3. **Существующие бонусы**: 
   - Бонусы, начисленные до 28.01.2025, сгорают через 1 год?
   - Или все бонусы получают срок 3 года с момента начисления?

4. **Подарочные сертификаты**:
   - Какой процент покрытия для сертификатов?
   - Зависят ли они от уровня ПЛ?

5. **Тестирование**:
   - Есть ли тестовая среда?
   - Нужны ли тестовые данные?

---

## SQL-скрипты для изменений

### 1. Обновление процентов покрытия

```sql
-- Сначала нужно узнать M_Promotion_ID для каждого уровня
-- Запрос для проверки текущих значений:
SELECT 
    pr.M_Promotion_ID,
    p.Name AS PromotionName,
    pr.RewardType,
    pr.Amount,
    pr.IsActive
FROM M_PromotionReward pr
JOIN M_Promotion p ON p.M_Promotion_ID = pr.M_Promotion_ID
WHERE pr.RewardType = 'B'
  AND pr.IsActive = 'Y'
ORDER BY pr.M_Promotion_ID;

-- После определения M_Promotion_ID для каждого уровня:
-- Level 1 (Знаток Вкуса) - обычно самый низкий ID
UPDATE M_PromotionReward 
SET Amount = 25,
    Updated = NOW(),
    UpdatedBy = 100
WHERE M_Promotion_ID = <Level1_Promotion_ID>  -- ЗАМЕНИТЬ на реальный ID
  AND RewardType = 'B' 
  AND IsActive = 'Y';

-- Level 2 (Гурман Вкуса)
UPDATE M_PromotionReward 
SET Amount = 50,
    Updated = NOW(),
    UpdatedBy = 100
WHERE M_Promotion_ID = <Level2_Promotion_ID>  -- ЗАМЕНИТЬ на реальный ID
  AND RewardType = 'B' 
  AND IsActive = 'Y';

-- Level 3 (Магистр Вкуса)
UPDATE M_PromotionReward 
SET Amount = 75,
    Updated = NOW(),
    UpdatedBy = 100
WHERE M_Promotion_ID = <Level3_Promotion_ID>  -- ЗАМЕНИТЬ на реальный ID
  AND RewardType = 'B' 
  AND IsActive = 'Y';
```

### 2. Изменение расчета BurnDate в представлении

```sql
-- ВАЖНО: Сделать бэкап представления перед изменением!
-- CREATE OR REPLACE VIEW adempiere.ip_loyalty_bonus_summary AS ...

CREATE OR REPLACE VIEW adempiere.ip_loyalty_bonus_summary
AS 
WITH sysvar_acc AS (
    SELECT to_number(COALESCE(ad_sysconfig.value, '0'::character varying)::text, '9999999'::text) AS acc_id
    FROM ad_sysconfig
    WHERE ad_sysconfig.name::text = 'BONUS_ACCOUNT_ID'::text
), 
sysvar_pt AS (
    SELECT ad_sysconfig.value
    FROM ad_sysconfig
    WHERE ad_sysconfig.name::text = 'BONUS_POSTING_TYPE'::text
)
SELECT 
    fas.c_bpartner_id,
    fas.userelement1_id AS bonuslot_id,
    sum(fas.amtsourcecr - fas.amtsourcedr) AS bonusamt,
    fas.c_currency_id,
    CASE
        WHEN ppc.m_promotionprecondition_id IS NOT NULL 
            THEN COALESCE(ppc.enddate, 
                CASE 
                    WHEN CURRENT_DATE < '2025-01-28'::date THEN CURRENT_DATE + '1 year'::interval
                    ELSE CURRENT_DATE + '3 years'::interval
                END)::date
        ELSE max((
            jl.dateacct::timestamp with time zone + 
            pt.netdays::character varying::interval + 
            CASE 
                WHEN jl.dateacct::date < '2025-01-28'::date THEN '1 year'::interval
                ELSE '3 years'::interval
            END
        )::date)
    END AS burndate,
    sum(fas.amtacctcr - fas.amtacctdr) AS ruramt,
    fas.m_promotionprecondition_id
FROM fact_acct_summary fas
JOIN gl_journalline jl ON jl.gl_journalline_id = fas.userelement1_id
JOIN c_paymentterm pt ON pt.c_paymentterm_id = jl.c_paymentterm_id
JOIN gl_journal j ON j.gl_journal_id = jl.gl_journal_id
LEFT JOIN m_promotionprecondition ppc ON ppc.m_promotionprecondition_id = fas.m_promotionprecondition_id 
    AND (EXISTS (
        SELECT 1
        FROM m_promotion prm
        WHERE prm.m_promotion_id = ppc.m_promotion_id 
          AND prm.c_campaign_id = 1001561::numeric
    ))
WHERE fas.postingtype = ((SELECT sysvar_pt.value FROM sysvar_pt)::bpchar) 
  AND fas.account_id = ((SELECT sysvar_acc.acc_id FROM sysvar_acc))
GROUP BY fas.c_bpartner_id, fas.userelement1_id, fas.c_currency_id, fas.m_promotionprecondition_id, ppc.m_promotionprecondition_id
ORDER BY fas.c_bpartner_id;
```

### 3. Проверка после изменений

```sql
-- Проверить новые проценты покрытия:
SELECT 
    pr.M_Promotion_ID,
    p.Name AS PromotionName,
    pr.Amount AS PercentCoverage
FROM M_PromotionReward pr
JOIN M_Promotion p ON p.M_Promotion_ID = pr.M_Promotion_ID
WHERE pr.RewardType = 'B'
  AND pr.IsActive = 'Y'
ORDER BY pr.M_Promotion_ID;

-- Проверить расчет BurnDate для тестового клиента:
SELECT 
    c_bpartner_id,
    bonuslot_id,
    bonusamt,
    burndate,
    CASE 
        WHEN burndate - CURRENT_DATE < 365 THEN '1 year'
        ELSE '3 years'
    END AS expiration_period
FROM ip_loyalty_bonus_summary
WHERE c_bpartner_id = <TEST_BPARTNER_ID>  -- ЗАМЕНИТЬ на тестовый ID
  AND bonusamt > 0
ORDER BY burndate;
```

## Следующие шаги

1. **Немедленно**: 
   - [x] Найти структуру представления - **СДЕЛАНО**
   - [ ] Определить M_Promotion_ID для каждого уровня ПЛ
   - [ ] Проверить текущие значения процентов покрытия в БД

2. **Подготовка**: 
   - [x] Составить SQL-скрипты - **СДЕЛАНО** (см. выше)
   - [ ] Протестировать скрипты на тестовой среде
   - [ ] Подготовить план отката изменений

3. **Тестирование**: Подготовить тестовый план:
   - Сценарии для каждого уровня ПЛ
   - Проверка расчета списания
   - Проверка расчета срока годности
   - Проверка API

4. **Документация**: Обновить:
   - Техническую документацию
   - Документацию API
   - Инструкции для пользователей

---

## Файлы для детального анализа

### Основные файлы:
1. `org.ipalich.loyalty/src/org/ipalich/loyalty/process/BonusMechanism.java`
   - Метод `getRewardAmount()` - получение процентов
   - Метод `calcBonusOnSales()` - расчет списания

2. `org.ipalich.loyalty/src/org/ipalich/loyalty/service/LoyaltyApiService.java`
   - Методы работы с процентами покрытия
   - Методы получения истории бонусов

3. `org.ipalich.webservices.rest/src/org/ipalich/loyalty/api/LoyaltyAPI.java`
   - API endpoints

### Дополнительные файлы:
- `BonusBurning.java` - процесс сгорания бонусов
- `SchedulerNotifyClientsFutureBonusBurning.java` - уведомления
- `CreateGLforBonus.java` - создание GL-проводок (возможно, там устанавливается BurnDate)

---
 
**Статус**: ✅ SQL-структура найдена, готовы скрипты изменений. Требуется определить M_Promotion_ID для уровней ПЛ.

## Дополнительная информация

### Структура представления `ip_loyalty_bonus_summary`

**Назначение**: Агрегирует данные о бонусах из `fact_acct_summary` и рассчитывает дату сгорания.

**Ключевые поля:**
- `c_bpartner_id` - ID контрагента
- `bonuslot_id` - ID партии бонусов (ссылка на `GL_JournalLine`)
- `bonusamt` - Сумма бонусов в партии
- `burndate` - Дата сгорания (рассчитывается)
- `m_promotionprecondition_id` - Ссылка на карту/сертификат

**Особенности:**
- Использует `C_PaymentTerm.netdays` (обычно 0 для бонусов)
- Для сертификатов может использовать `ppc.enddate`
- Группирует по партиям бонусов (`userelement1_id`)

### Функция `ip_loyalty_get_bonus_burn()`

**Назначение**: Определяет, какие бонусы нужно списать (сгорели).

**Логика:**
- Использует представление `ip_loyalty_bonus_summary`
- Фильтрует бонусы где `BurnDate <= CURRENT_DATE`
- Исключает бонусы, которые находятся в холде (операции типа 'HLD', 'HLC')
- Учитывает кумулятивные суммы для правильного списания

**Важно**: Функция не требует изменений, т.к. работает с результатом представления.

