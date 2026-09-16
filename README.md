# Ukraine Stores · Monobank і фінальне списання після збірки

**Статус:** internal working doc · для узгодження з Payments, Product і Monobank  
**Автор / контакт:** Yuliia Nikolaieva  
**Оновлено:** 16 Sep 2026  
**Контекст у Slack:** [тред C0BD2EQR4LW](https://taxify.slack.com/archives/C0BD2EQR4LW/p1788978109541509) *(потрібно доповнити фактами з обговорення)*

**Пов’язані матеріали:**

| Матеріал | Посилання |
|----------|-----------|
| Business case «друга транзакція» (EN/UA) | [ukraine-second-payment-business-case](https://github.com/yuliianikolaieva/ukraine-second-payment-business-case) |
| Monobank agency / split (LIKI24 + BOND) | локально: `Bolt & Mono/bolt-mono-flow (1)_EN.html` |
| Legal review overbooking / second charge | [LSS-64024](https://taxify.atlassian.net/servicedesk/customer/portal/47/LSS-64024) |
| Monobank API (split invoice) | [monobank.ua/api-docs](https://monobank.ua/api-docs/acquiring/methods/split/post--api--merchant--invoice--create) |

---

## Коротко (TL;DR)

Після збірки кошика в grocery **фінальна сума Y часто відрізняється** від того, що eater сплатив на checkout **X** (заміни, вага, пакування). Ринковий стандарт UA (зокрема Glovo) — **списати фінальний кошик після picking**. Bolt Stores зараз списує **один раз до збірки**, і Bolt закриває різницю Y−X.

**Цільовий патерн для UA** — не «тихе дописування» як у Glovo, а **rides-style**: estimate → hold → показати фінальний кошик → consent де потрібно → **release hold → фінальне списання** ([business case](https://yuliianikolaieva.github.io/ukraine-second-payment-business-case/)).

**Monobank на рівні API** вміє двофазну оплату (hold / finalize / cancel) і сценарій «скасувати hold → списати повну суму по збереженому токену», але **це не те саме**, що вже працює в Bolt Ride Hailing на глобальному payment stack, і **Bolt Food Stores сьогодні цього не вміє «з коробки»** — потрібна окрема product + integration інвестиція, плюс compliance UX, плюс узгодження agency/split для масштабу grocery.

---

## 1. Що нам потрібно (бізнес-вимога)

Після прийняття замовлення і збірки в магазині:

1. **Авторизувати / заблокувати** орієнтовну суму кошика (estimate), з прозорими правилами для вагових товарів і замін.
2. Отримати **фінальний кошик** (кількості, заміни, пакування).
3. **Показати eater deltas** і отримати **згоду**, якщо ціна зростає (особливо дорожчі заміни) — вимога Legal ([LSS-64024](https://taxify.atlassian.net/servicedesk/customer/portal/47/LSS-64024)).
4. **Зняти попередній hold** (release).
5. **Списати фінальну суму Y** однією узгодженою транзакцією (з коректним розподілом комісії Bolt / виплатою партнеру, де застосовується agency scheme).

**Чого ми свідомо не хочемо:** Glovo-style *silent upcharge* — одна «фіксована» ціна на checkout і потім більше списання без чіткого disclosure у флоу (Legal це категорично не рекомендує).

**Навіщо це критично:** без фінального списання Bolt фінансує **settlement gap Y−X**, втрачає **service fee** на дельті, тягне **ops-податок** (Retail / Finance / пікери) — див. цифри в [business case](https://yuliianikolaieva.github.io/ukraine-second-payment-business-case/).

---

## 2. Як це працює в Bolt Ride Hailing (референс)

Ride Hailing у Bolt уже побудований навколо **оцінки → hold → фінальна сума після послуги**:

- Клієнт бачить, що сума на старті — **estimate**, а не остаточний чек.
- Після поїздки застосовується **фінальна ціна**; клієнт бачить settled fare.
- Legal вважає такий підхід **комплаентним**, якщо disclosure і фінальна сума прозорі — на відміну від «фіксованого» checkout у Stores з подальшим дописуванням.

**Важливо:** RH і Stores — **різні вертикалі в payment orchestration**. Те, що RH «вміє» на глобальному стеку, **не означає**, що той самий флоу вже підключений до **Monobank agency/split** для Food Stores на всіх партнерів UA.

---

## 3. Що Monobank підтримує (за нашою технічною схемою LIKI24 / BOND)

За внутрішнім описом інтеграції Bolt × monobank ([`bolt-mono-flow`](../../Bolt%20&%20Mono/bolt-mono-flow%20(1)_EN.html)):

| Сценарій | Поведінка API (спрощено) |
|----------|---------------------------|
| **Y = сума hold** | `invoice/finalize` на повну суму hold → split: комісія Bolt, залишок sub-merchant |
| **Y < hold** | Частковий `finalize` на потрібну суму; решта повертається клієнту |
| **Y > hold** | `invoice/cancel` (повне скасування hold) → `wallet/payment` на **повну** суму Y по збереженому `cardToken` (MIT, без повторного вікна оплати) |
| **Токенізація** | `saveCard` разом з hold; hold **до 9 днів**, інакше автоматичне скасування |
| **Agency / split** | `splitReceiverId`, `agentFeePercent`; один платіж з автоматичним розподілом |

Тобто Monobank **не «відмовляється» від двофазної оплати взагалі** — для пілотів pharmacy / BOND сценарії описані явно.

---

## 4. Що Monobank **не** дає (або дає обмежено) для масштабного Stores MVP

Нижче — формулювання для обговорення з Monobank і Payments. Частину пунктів треба **підтвердити** у треді Slack / на дзвінку з банком.

### 4.1. Немає «дописати на той самий hold» (incremental capture)

Якщо фінальна сума **вища за hold**, Monobank у нашій схемі вимагає **повного cancel** поточного інвойсу і **нового** списання на всю суму Y (через `wallet/payment`), а не класичного *incremental authorization* / capture delta на тій самій авторизації, як на деяких глобальних еквайєрах.

**Наслідок для Bolt:** два кроки в оркестрації, ризик **подвійного відображення** в історії картки (release + нове списання), жорсткіші вимоги до **MIT / consent** і моніторингу відмов другого списання.

### 4.2. Agency split на «другому колі»

Перший hold створюється з конкретним `splitReceiverId` і комісією. Після cancel + нового charge треба **знову** коректно провести split на **нову** суму Y (і для grocery — **багато** sub-merchants / магазинів, не один LIKI24).

**Питання масштабу:** чи покриває поточна monobusiness / agency угода **тисячі store-level receivers** і той самий SLA, що для RH-обсягів?

### 4.3. Термін hold 9 днів

Для типового grocery delivery це зазвичай достатньо, але це **жорстка межа** порівняно з деякими міжнародними схемами; edge cases (затримки, спори) треба явно обробляти.

### 4.4. Відмінність від «як у Glovo» на картці клієнта

Технічно сценарій «спочатку повернули 100 €, потім зняли 120 €» **може** бути реалізацією cancel hold + full capture (див. питання 1 нижче). Це **не** означає, що Bolt може повторити Glovo **без** rides-style disclosure і consent — Legal boundary інша.

### 4.5. Bolt Product / Stores tooling

Навіть за наявності API, **поточний Stores payment flow** не реалізує `hold → release → final charge` end-to-end (конфіг overbooking це не замінює) — цитата Product/tech у [business case](https://yuliianikolaieva.github.io/ukraine-second-payment-business-case/):

> *Second payment / UA market standard is not a config fix — Product must decide whether to invest; current tooling cannot deliver hold → release → final charge out of the box.*

---

## 5. Чому ми **не можемо просто зробити як у Ride Hailing** (сьогодні)

| Фактор | Ride Hailing | Ukraine Stores (зараз) |
|--------|--------------|-------------------------|
| **Payment orchestration** | Зрілий RH-флоу estimate / final на глобальному стеку | Один capture на checkout; немає готового UA Stores MVP |
| **Еквайєр / схема** | Інша інтеграція, ніж Monobank agency split для Food | Monobank — фокус для agency (LIKI24, BOND); grocery at scale — окреме рішення |
| **Отримувач платежу** | Модель RH | Agency: клієнт бачить **Bolt**, split на sub-merchant |
| **Compliance UX** | Estimate заведено в продукті | Потрібен новий checkout / post-pick UX + item-level consent |
| **Операційна модель** | Фінальна ціна після поїздки — норма | Партнери очікують фінальний акт Y; зараз Bolt латає Y−X |

**Висновок для стейкхолдерів:** «як у RH» — це **правильний цільовий патерн** (і Legal його підтримує для Stores MVP), але **не кнопка в адмінці**. Потрібні: product investment, Monobank/Payments sign-off, інтеграція split на фінальному charge, QA сценаріїв X&lt;Y, X&gt;Y, відмова другого списання.

---

## 6. Відкриті питання (для Monobank, Payments, Legal)

### 6.1. Як Monobank підтримує сценарій Glovo?

**Спостереження з ринку (приклад):** клієнту спочатку блокують / списують **100 €**, після збірки факт **120 €** — на картці виглядає як **повернення 100 €** і потім **списання 120 €**.

**Питання до Monobank / Payments:**

1. Чи це стандартна реалізація через **cancel hold + MIT `wallet/payment`** на повну суму, чи інший продукт (окремий incremental auth)?
2. Чи Glovo в UA ходить через **той самий** monobank acquiring / agency split, що ми плануємо для Bolt Food?
3. Які вимоги до **згоди клієнта** на MIT при сумі **вищій** за первісний hold?
4. Як виглядає **chargeback / dispute** policy, якщо клієнт оскаржує друге списання?
5. Чи підтримується **той самий** `cardToken` після cancel без повторного 3DS?

### 6.2. Райффайзен Банк Аваль

У Bolt є відносини / рахунки з **Райффайзен Банк Аваль** (уточнити: acquiring, settlement, не лише корпоративний рахунок).

**Питання:**

1. Чи є в Авалі **еквайринг** з двофазною оплатою та **incremental authorization** або capture adjustment на одній авторизації?
2. Чи підтримується **marketplace / split** під модель Bolt Food (комісія + виплата мерчанту) на обсязі grocery?
3. Чи швидше time-to-market **розширити Monobank** vs **паралельний PSP (Аваль)** для Stores second payment?
4. Комісії, SLA, chargeback — порівняльна таблиця Monobank vs Аваль для цього use case.

### 6.3. Чому не клонувати RH payment flow один-в-один?

**Питання до Global Payments / Product:**

1. Який **PSP і merchant setup** обслуговує RH в Україні vs Stores checkout сьогодні?
2. Чи можна **переюзати** RH payment rails для vertical=Stores без нового MID / договору / split logic?
3. Який **мінімальний MVP scope** (країна, партнери, % GMV) для hold → final charge?
4. Що блокує в **backlog** (order lifecycle, picker events, refunds, reconciliation з актами партнерів)?

---

## 7. Запропоновані наступні кроки

1. **Payments:** підтвердити фактичний RH vs Stores stack в UA (таблиця 1 сторінка).
2. **Monobank:** воркшоп по сценаріях X=Y, X&lt;Y, X&gt;Y на **production-like** agency split; письмові відповіді на питання §6.1.
3. **Legal + Product:** зафіксувати UX MVP (estimate, фінальний кошик, consent) — узгоджено з [LSS-64024](https://taxify.atlassian.net/servicedesk/customer/portal/47/LSS-64024).
4. **Finance / Retail:** правила reconciliation, коли hold скасовано, а друге списання failed.
5. **Опційно:** RFI до **Райффайзен Аваль** (§6.2) — deadline і owner від Finance/Payments.

---

## 8. Що доповнити з Slack-треду

Після перегляду [повідомлення в Slack](https://taxify.slack.com/archives/C0BD2EQR4LW/p1788978109541509) варто вставити сюди:

- [ ] Точні цитати / позиція Monobank (що саме «не підтримують»)
- [ ] Ім’я та роль контакту з банку
- [ ] Домовленості / timeline
- [ ] Блокери з боку Bolt (якщо в треді названі окремо від банку)

---

*Internal use only · Bolt Food Ukraine · Stores payments*
