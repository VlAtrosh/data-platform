# Практикум: explain и индексы

**Дата:** 2026-10-05
**База:** `logs.events` (1200 документов)

---

## Сводная таблица планов

| #   | Задача               | Индекс                                  | Порядок | nReturned | DocsExamined | KeysExamined | Шаги плана             |
| --- | -------------------- | --------------------------------------- | ------- | --------- | ------------ | ------------ | ---------------------- |
| 1   | Запрос 1 без индекса | —                                       | —       | 33        | 1200         | 0            | COLLSCAN → SORT        |
| 2   | Запрос 1, ESR        | `service_1_level_1_ts_-1_duration_ms_1` | E, S, R | 33        | 33           | 43           | IXSCAN → FETCH         |
| 3a  | Вариант А            | `duration_ms_1_service_1_level_1_ts_-1` | R, E, S | 33        | 33           | 378          | IXSCAN → SORT → FETCH  |
| 3b  | Вариант Б            | `service_1_level_1_duration_ms_1_ts_-1` | E, R, S | 33        | 33           | 33           | IXSCAN → SORT → FETCH  |
| 4   | Запрос 2 без индекса | —                                       | —       | 10        | 1200         | 0            | COLLSCAN → SORT        |
| 4   | Запрос 2 с индексом  | `level_1_duration_ms_-1`                | E, S    | 10        | 10           | 10           | IXSCAN → FETCH → LIMIT |

**Запрос 1:**

```javascript
db.events
  .find({
    service: 'payments',
    level: 'error',
    duration_ms: { $gt: 500 },
  })
  .sort({ ts: -1 });
```

**Запрос 2:**

```javascript
db.events.find({ level: 'error' }).sort({ duration_ms: -1 }).limit(10);
```

---

## Задача 1 · Запрос без индекса

| Показатель          | Значение        |
| ------------------- | --------------- |
| `nReturned`         | 33              |
| `totalDocsExamined` | 1200            |
| `totalKeysExamined` | 0               |
| Шаги                | COLLSCAN → SORT |

**Ссылка на разбор:** https://dfrancour.dev/tools/mongodb-paste-the-plan#v1:7VZLc-M2DP4rGcweemAyUmzHiW7xq5tJFXsjbx-TzXgQEXK4kUgtSTnyZvzfO6T8TNtkj-1MTxJBfCAAfgD4AlSXOQr5K2kjlIQIQmDwrSK9nOQoJWmIXkBiQabElCCCXM3NCS1IWgMMhORUj0RuSSdkIcowN8SgRG2If3JmHP4DSg7R3QvktKDcS-gbREBaKw2rFXsBQ3oh3AGbvRKXhT_Eb_NKoxVKzgrjVeYWok4QrFb3q7W7H9E8QgT9XtjuDM8HwKDMUfYxfaRrWkIEg1EvDMNhDxio0opCfPcWp6KgWOS5MBCdMSiwvnIxER_rROWVUzG35MzwbXg7pUvJ39JKUpRmqoZ1mStOrxWehZRCzl2iXVDG4txleDSc9j_63JaVTRrhbvfq96R_eQMMnmg5QWtJN-BN_kK2yXLIwBqIjkN2mL5wtb63GyycxTV0Fs48cBbOrJkdh7M90MyxQpi4yq3w2dyEuBZM0D6aAzfu7rd-uF_niPseOHLnLk-Yz1J8q2hrU5jE02dPMEFtBeY7ifN-S9lTBlxoSm1D4EzpZ9R8Q86eqiR_5Rvcfdny6wuwo_3VPex57hQ9SRut9a9X8RHBXYz1NS3ZUSzkNS39zmGM8FMnCNiRkNlJcA_3q9WKgaavlFri7uY3eaCaUs-jxKL1_u4kVZqSMRBZXREDeUu20tIRqdXaA-4zucvAKov5NS3NsMZCePV2ay0eqHRPfGDFE878HR_fP3dorCjQEkQBg2eln0xzJvIFynQDlETcYSC6aBZ_CMq5xxhckIu_saDJWKX3BMIMxyNPbP6XADDXhNy1gfHD10b77fr5N4Xzfy3_J2vZX9yr-jJEjidh4MClmZKx1LDBLQdalaVf-0aQqqLws_EFMuG-sJ2smR-qB7HuZuI2qvUIZf88IBkYpa2TNQRaMfjAH9aDHNw26QXpK5kpp_SojHXDcphcT8eT44ub6c3lbddNU2_ltBuEXQaL7XOhexKctFvAYC7s7hFBWQe7vNU-Dx46FxkhhS3eOWu3umfh6UWWtlMiOgu73d35E9RYkCXt3RfS1QLm_gUxwpRsr8oy0on4Tr2lJZ_g9nmnexb4Qn-tHWM9rmxZ2YFK38b8otRTVfouceVEBXGBlgYqrVyqY6zfxm8UE1XplH7WqipjrGMqlF6-42mMdS9X6ZOQ80Rp22A-G5y_F-JEq0fxIOwGHZOe01jGSs5Vsm59h-dccj5VCdkfjSMh-5uQXD2PXC8zPxrRyN2ha5R9Ja1WeVPBKfVzNEakQzkXktydqyfXuv4E

**Почему сервер прочитал 1200 документов, чтобы вернуть 33:**

Индекса по полям `service`, `level`, `duration_ms` нет. MongoDB вынуждена перебирать всю коллекцию (`COLLSCAN`), проверяя фильтр на каждом документе. Сортировка (`SORT`) выполняется в памяти отдельным шагом.

---

## Задача 2 · Индекс по правилу ESR

**Разбор запроса:**

- **E (Equality):** `service: "payments"`, `level: "error"`
- **S (Sort):** `ts: -1`
- **R (Range):** `duration_ms: { $gt: 500 }`

**Команда создания индекса:**

```javascript
db.events.createIndex({ service: 1, level: 1, ts: -1, duration_ms: 1 });
```

| Показатель          | Значение       |
| ------------------- | -------------- |
| `nReturned`         | 33             |
| `totalDocsExamined` | 33             |
| `totalKeysExamined` | 43             |
| Шаги                | IXSCAN → FETCH |

**Ссылка на разбор:** (https://dfrancour.dev/tools/mongodb-paste-the-plan#v1:7VZLc-M2DP4rGcweemAyUmzHiW7xq5tJFXsjbx-TzXgQEXK4kUgtSTnyZvzfO6T8TNtkj-1MTxJBfCAAfgD4AlSXOQr5K2kjlIQIQmDwrSK9nOQoJWmIXkBiQabElCCCXM3NCS1IWgMMhORUj0RuSSdkIcowN8SgRG2If3JmHP4DSg7R3QvktKDcS-gbREBaKw2rFXsBQ3oh3AGbvRKXhT_Eb_NKoxVKzgrjVeYWok4QrFb3q7W7H9E8QgT9XtjuDM8HwKDMUfYxfaRrWkIEg1EvDMNhDxio0opCfPcWp6KgWOS5MBCdMSiwvnIxER_rROWVUzG35MzwbXg7pUvJ39JKUpRmqoZ1mStOrxWehZRCzl2iXVDG4txleDSc9j_63JaVTRrhbvfq96R_eQMMnmg5QWtJN-BN_kK2yXLIwBqIjkN2mL5wtb63GyycxTV0Fs48cBbOrJkdh7M90MyxQpi4yq3w2dyEuBZM0D6aAzfu7rd-uF_niPseOHLnLk-Yz1J8q2hrU5jE02dPMEFtBeY7ifN-S9lTBlxoSm1D4EzpZ9R8Q86eqiR_5Rvcfdny6wuwo_3VPex57hQ9SRut9a9X8RHBXYz1NS3ZUSzkNS39zmGM8FMnCNiRkNlJcA_3q9WKgaavlFri7uY3eaCaUs-jxKL1_u4kVZqSMRBZXREDeUu20tIRqdXaA-4zucvAKov5NS3NsMZCePV2ay0eqHRPfGDFE878HR_fP3dorCjQEkQBg2eln0xzJvIFynQDlETcYSC6aBZ_CMq5xxhckIu_saDJWKX3BMIMxyNPbP6XADDXhNy1gfHD10b77fr5N4Xzfy3_J2vZX9yr-jJEjidh4MClmZKx1LDBLQdalaVf-0aQqqLws_EFMuG-sJ2smR-qB7HuZuI2qvUIZf88IBkYpa2TNQRaMfjAH9aDHNw26QXpK5kpp_SojHXDcphcT8eT44ub6c3lbddNU2_ltBuEXQaL7XOhexKctFvAYC7s7hFBWQe7vNU-Dx46FxkhhS3eOWu3umfh6UWWtlMiOgu73d35E9RYkCXt3RfS1QLm_gUxwpRsr8oy0on4Tr2lJZ_g9nmnexb4Qn-tHWM9rmxZ2YFK38b8otRTVfouceVEBXGBlgYqrVyqY6zfxm8UE1XplH7WqipjrGMqlF6-42mMdS9X6ZOQ80Rp22A-G5y_F-JEq0fxIOwGHZOe01jGSs5Vsm59h-dccj5VCdkfjSMh-5uQXD2PXC8zPxrRyN2ha5R9Ja1WeVPBKfVzNEakQzkXktydqyfXuv4E)

**Во сколько раз уменьшилось число просмотренных документов:**

1200 / 33 ≈ **36 раз**.

**Почему из плана пропал шаг SORT:**

Поле `ts: -1` стоит в индексе после равенств `service` и `level` (правило ESR). MongoDB читает участок индекса подряд — внутри него документы уже отсортированы по `ts`. Отдельный шаг SORT в памяти не нужен.

---

## Задача 3 · Порядок полей

| Индекс                            | Порядок | Возвращено | Документов просмотрено | Ключей просмотрено | Шаги плана            |
| --------------------------------- | ------- | ---------- | ---------------------- | ------------------ | --------------------- |
| нет                               | —       | 33         | 1200                   | 0                  | COLLSCAN → SORT       |
| `service, level, ts, duration_ms` | E, S, R | 33         | 33                     | 43                 | IXSCAN → FETCH        |
| `duration_ms, service, level, ts` | R, E, S | 33         | 33                     | 378                | IXSCAN → SORT → FETCH |
| `service, level, duration_ms, ts` | E, R, S | 33         | 33                     | 33                 | IXSCAN → SORT → FETCH |

**Ссылки на разборы:**

- Вариант А: (https://dfrancour.dev/tools/mongodb-paste-the-plan#v1:7Vbbbts4EP2VYNCHfWACKb4leotjexukjr2RuxekgTERxwpriVRJyrEb6N8XpHxL0yTd3ZddYJ8kDueQZ4YzB_MItCwyFPJX0kYoCRGEwOBLSXo1zlBK0hA9gsScTIEJQQSZSs0RLUhaAwyE5LQciMySjslCNMPMEIMCtSH-izvG4d-h5BDdPEJGC8q8hb5ABKS10lBV7BEM6YVwF2z2Clzl_hK_zUuNVig5zY13SS1ErSCoqttqTfc9mnuI4LwbNlv9kx4wKDKU55jc0yWtIILeoBuGYb8LDFRhRS6--hMnIqehyDJhIAoY5Li8cDERH-lYZaVzMdfkjuHb8HZOZ5K_5hUnKM1E9ZdFpjh96_AgpBQydYl2QRmLqcvwoD85f-9zW5Q2ro273Xh0PQEGRmk7RmtJe6w1EB2GFYOc8g8iFxaiMGietDrtIGBgV4WDcpphmdkXj774PT4_uwIGc1rtnf0k9yHbPVXINg8ash0DXxJXmPsbd9BpOF0Dp-HUw6bh1JrpoSs4YYZlZoV_qE321oYx2nvzjMbN7R4Pt1gTcb_W7zsm5qMUX0ranilM7CtzzzBGbQVmO4tjv-2GYwZcaEps3RszpR9Q803dd1Up-Xe4wU-tIGAHQs6Oglt4whRuPm0r-xOwg_2Vd93E4Rx9e9Re61_v4uODmyEuL2nFDoZCXtLqFm6rqqoYaPpMiSXuqmqTCFpS4ms0tmg94Z2lTBIyBiKrS2Igr8mWWroibTT2gPtdcszAKovZJa1Mf4m5qN07J2t7TyX79sbT-1My3yv2ty_uGytytARRg8GD0nMDUTNsMEC-QJlskJKIO5BjdFov_xCUcd_dBhfkckB-pclYpfcMwvRHA1_N_FkMmGlC7mRmdPe59n6tP_9l8fwzuaifFS3G4ivFSltPLmyfMCgN8Z4w820DmUJkWa2l_pe4B1qlMSWHfyV3WwH6K9kLttnzCXoxe83W387e_3r4n9RD_3DfSpQh8rXSbDPgZWEmZHw5B_Wyp1VR-HUtp4nKcz-9PMJMuC9sZ5-ZH3t8_W7i2U0tW-brIYe9PMLU3fmkLd_xu_WoBVWdLtIXcqac070y1o0z_fhyMhofnl5Nrs6uO27e8accd4Kww2CxHeg6R8FRswEMUmF3Yx7NWtjhjeZJcNc6nRFS2OCtdrPRaYfHp7OkmRBRO-x0dvePUWNOlrSnL6RrB8z8jDfAhGy3nM1Iuw7vrqwT-T1Fee49xOWotEVpeyp5HfNBqXlZeK24cKacuEBLPZWULtVDXL6O3zjGqtQJ_axVWQxxOaRc6dUbTIe47GYqmQuZOtWrMR8Npm-FONbqXtwJu0EPSac0kkMlUxWvBfDpPWecT1RM9kfjiMn-JiRXDwMnZ-ZHIxq4N3Ryea6k1Sqrmzih8wyNEUlfpkKSe3M1hyis_gQ)
- Вариант Б: https://dfrancour.dev/tools/mongodb-paste-the-plan#v1:7Vbdb-I4EP9XqtE-3ENaJQVKm7dS4LbqUtiGvQ91KzSNh9SLY2dth8JW_O8nOxCge21Xey930j0lHs9v_JtPzRPQohDI5W-kDVcSYogggK8l6eVIoJSkIX4CiTmZAlOCGITKzBHNSVoDAXDJaNHnwpJOyEI8RWEogAK1IfbRmXH4dygZxLdPIGhOwkvoK8RAWisNq1XwBIb0nLsHNncFLnP_iL9mpUbLlZzkxqtkFuJWGK5Wd6s13fdoHiCGi07UbPVOuxBAIVBeYPpAV7SEGLr9ThRFvQ4EoArLc_7NWxzznAZcCG4gDgPIcXHpfCI21IkSpVMxN-TMsNq9rdK5ZK9pJSlKM1a9RSEUo-cKj1xKLjMXaOeUsZi5CPd744v3PrZFaZNKuL1NhjdjCMAobUdoLWmPtQbiw2gVQE75B55zC3EUNk9b7ZMwDMAuCwdlNMVS2BdNX_6RXJxfQwAzWu7YrlMTBZsERsF-RqJgy8CXxDXmzuIaOokmHjiJJjuwSTSxZnLoCo6bQSks94naRG8tGKF9MHs0bu9qHu53j4gTWP91TMwnyb-WVNvkJvGVuSMYobYcxVbi2NfdcBwA45pSW_XGVOlH1GxT9x1VSvaMG9x-rkv3MwQHu6c72GHuFH39V1rrX6-y7xH80grD4IDL6VHor71_cDvAxRUtg4MBl1e0vIO71Wq1CkDTF0otMVdVm0DQglJfo4lF6wlvJWWakjEQW11SAPKGbKmlK9JGYwf4rEussiiuaGl6C8x5re7FXZU-E-8-n5H5u1p_-92esTxHS_79R6VnBuKT0wCQzVGmG6AkYg4DcaNZnf7kJJgHGZyTC0BlQpOxSu8IuOkN-1Vpf-cBCk3I3IwZ3n-ptF9rzn-VN_9sUlQpRYsJ_0aJ0tZzixzX0hDrcjOre8cUXIiqQPwvMQ-0SmNGDv9K5OrZ83Oxc_F5KXbhT4fu_zn4n5yDPnHPOtgQzcw6YYUZk_GFHFbHrlZF4c_VDE1VnvuV5Qmm3H2hXnimftfZ83W7qtRerTeb4OW9perLvYZ8x-7X-xW4a9Jz0pdyqpzSgzLW7TC95Go8HB2eXY-vz2_absnxVo7bYdQOYF5vce2j8KjZgAAybre7HU1b2GaN5ml43zqbElLUYK2TZqN9Eh2fTdNmSkQnUbu9fX-EGnOypD19Ll0voPCLXR9Tsp1yOiXteruztG6078yS77UHuBiWtihtV6WvYz4oNSsLPyUunSgnxtFSV6WlC_UAF6_jN4qJKnVKv2pVFgNcDChXevkG0wEuOkKlMy4zN-8qzCeD2VsujrR64PfcbtAD0hkN5UDJTCXr0bf_zjljY5WQ_VE_ErK_c8nUY9_NMvOjHvVdDt2gvFDSaiWqDk7pQqAxPO3JjEtyOVcziKPVXw

**Почему в варианте А просмотрено столько ключей (378):**

Первое поле индекса — `duration_ms` (диапазон `> 500`). MongoDB не может сразу перейти к `service: "payments", level: "error"` — она читает все события с `duration_ms > 500` (все сервисы, все уровни). 378 ключей — это все события с `duration_ms > 500`. Только 33 из них — `payments + error`. Index Efficiency: 8.7%.

**Почему в варианте Б остался шаг SORT:**

Поле `ts` (сортировка) стоит после `duration_ms` (диапазона). Внутри участка, отобранного диапазоном, порядок по `ts` не сохраняется — индекс не может обеспечить готовую сортировку. Сервер досортировывает результат в памяти (шаг SORT). In-memory Sort?: Yes.

**Вывод — какой вариант оставили бы в рабочей базе:**

ESR (`service_1_level_1_ts_-1_duration_ms_1`). Хотя вариант Б просматривает меньше ключей (33 vs 43), в нём есть SORT в памяти. Сортировка ограничена 100 МБ — на реальном журнале запрос упадёт с ошибкой. ESR даёт готовую сортировку из индекса (In-memory Sort?: No) — масштабируется. Правило ESR подтверждено.

---

## Задача 4 · Самостоятельно

**Запрос:**

```javascript
db.events.find({ level: 'error' }).sort({ duration_ms: -1 }).limit(10);
```

**Мой индекс:**

```javascript
db.events.createIndex({ level: 1, duration_ms: -1 });
```

**Объяснение порядка полей:**

- **E (Equality):** `level: "error"` — первым (точное равенство)
- **S (Sort):** `duration_ms: -1` — вторым (сортировка)
- **R (Range):** нет
- По правилу ESR: **E → S → R**

| Показатель            | Без индекса | С индексом |
| --------------------- | ----------- | ---------- |
| `totalKeysExamined`   | 0           | 10         |
| `totalDocsExamined`   | 1200        | 10         |
| `executionTimeMillis` | 31ms        | 12ms       |
| `Has Sort?`           | Yes         | No         |

**Ссылки на разборы:**

- Без индекса: (https://dfrancour.dev/tools/mongodb-paste-the-plan#v1:7VXbcho5EP2VVFcexy6Gqz1vXDcUGUM8ZJMth3K1Rw0oaKSxpMGQFP--JXEN3rWzl4d92CdQz2n15Rx1fwda5QK5_JW04UpCBCEE8FiQXo8ESkkaou8gMSOTY0oQgVAzc0lLktZAAFwyWvW4sKQTshBNURgKIEdtiH1w1zh_QUsS7s9beoQISGulYbPZBXqHZg4R1LrleqVa60IAuUDZxnROA1pDBK1Kt9O6qncgAJVbnvFvaLmSY55RzIXgBqJSABmu-i4bYkOdKFE4iLkldw07JHYENSV7CZWkKM1YdVe5UIzOAU9cSi5nrkWuLGNx5nrzvh_3xxCA4Bm3zUwV0kIUllyb8sImW9QR3uuO2-_gz772Pyft5g0EsKD1CK0lLU96GQbACu37cJ8ZiC7CzY6NG8w8Tw53H96foO4vHLncxIWw3Ld2X-_OMEI7NydB7iZnUe4mLor5KPljQQd3bhJP-IlhhNpyFEeLy-wgsnIAjGtK7VZyU6WfULO9nFqqkOyHNODuy1Y0XyB4c_g7gWfpwV2MqwGtgzcxlwNaT2Cy2TihafpKqSXmGNvXQStKPf-JRevjHS1FmpIxEFldUADylmyhpROAY_MAO1VgWA7AKotiQGvTXWHGD3hv7qj0zHwaf0bmj4T0euCusTxDSxBVAnhSeuFSCQNAtkSZ7h0lEXM-_qG4w2-cBPMng0tyDdh-02Ss0icGbrrDnpfbXxT130y99K-nXgqAPes9Ck3I3OgZPnzdol9-hP-lcv4fCP9oIPgGnunBEHm-nH9uxmQsbUlxx45Wee7P22GSqixDyVxCU-5-4bAPp34VnqS6X3YBGKWt-_CcJf-ytnm8ZQ-7HetdSC9J9-VUOce5MhYi6HSTwXg4uri-Gd80bxtuXfqby41S2AhgedjkjcvSZbUCAcy4Pe53mtawwSrVq9JD7XpKSGGF1erVSqMelq-naTUlonrYaBzjj1BjRpa0p4BLJzwUfrn3MCXbKqZT0gn_Rq21Ja_66lWtUS_5V3WOjnE1LGxe2I5KX_Z5r9SiyP2T7DtTRoyjpY5Ki4yku-hl_z0wUYVO6RetijzGVUyZ0utXMo1x1RIqXXA5S5S2W5-PBmevlTjSas4fuN17x6RnNJSxkjOV7ObMj3GajI1VQvZn60jIfuKSqaeeGxzmZyvqOQ7dVGorabUS21eWUlugMTztyhmX5DhXC4jCze8)
- С индексом: (https://dfrancour.dev/tools/mongodb-paste-the-plan#v1:tVTbbuM2EP2XwT5qA0m-Rm-OL22QOPZG7i6KoigYcqSwpkgtOXLkXfjfC0rxrQmSbdE-WR7O4cycczjfAetSMak_o3XSaEggggC-Vmi3S8W0RgvJd9CsQFcyjpCAMrm7wA1qchCA1ALrmVSENkWCJGPKYQAlsw7FJ3-NxyvcoPIfH_ArJIDWGgu73XOhn5l7hAR607jf6famEECpmB4z_og3uD0_MSXJQn5jJI1eyQLnUinpIInCAApWX_t2UCxsalTlc9w9-nvEobNj0kiLt7JSzrRbmWldKiPw7wlPUmupc8-Rn8sRyz056eJ-BQE4Y2nJiNA2p6KyTcN_FA6Sj9EugAKLW1lI8o13h71BPwwDUD4yKkylqR2ItqW_1MmiVNiQXVaUtqWONceL29t0PLqDALJGiDcZF9Iip1bqzNgnZkUTt_gnckLhJ3KQ_Pb7LgCskTf8pMTI-duOkYpzdA4SshUGoO-RKqs9Qb7xQ9qpRJ0oADLE1A1u3bRmhWzyw-foxPCTaBSH4XkDObpXmH6_8NSRLBghJFE_gCdj194ucRQHwMSGab7HakThYU3xqP3_q0TV9ujYBj0NCEns2XJk7ElAuuliBkn0v2nfcsSIpfIbpsaSbzsexp0AKodiIt36YE5XSqVc27b_RNEAyViWo8c3R_-Fm07pH_4j_j3DJ_wPzwUI4_BfC_CawwMQL-y18wNwUxRMCz9bJv0vHFbbi_H3c7cavy6u2isbwAfx8LwuGwjaDdprnRkPfDSOIIHJNL1ZLZYfL-9Wd6P7gd98zc3xIIwGAWwOS3lwEV50OxBALum4qjHrsYHodIfhQ-8yQ4ZRR_T63c6gH8WXGe9yROxHg8Gx_pJZViChbd6S1N6lTDV7esY40lWVZWi9Ra62hO7Moi-z56xeVFRWNDH8bcytMeuqbMx27UMFCskIJ4ZXBWp_0dv4fWJqKsvxJ2uqcs7qORbGbt_pdM7qK2X4WurcP5sW84tj-XsjLq15lA-S9ug52hwXem50btLnF3ReZyTEyqRIPzpHivRFamGeZt7l7kcnmnkN_UMaG03WqNbnHMeKOSf5VOdSo9fcrCGJdn8B)

**Почему в коллекции 180 ошибок, а сервер прочитал только 10 документов:**

Индекс `level_1_duration_ms_-1` упорядочен по `level`, внутри — по `duration_ms: -1`. MongoDB идёт по индексу, берёт первые 10 документов с `level: "error"` (самые долгие), останавливается (LIMIT). Не нужно читать все 180 ошибок — только 10.

---

## Ответы на вопросы приёмки

**Чем `totalKeysExamined` отличается от `totalDocsExamined`:**

`totalKeysExamined` — сколько записей индекса просмотрено. `totalDocsExamined` — сколько документов прочитано из коллекции. Индекс может просмотреть больше ключей, чем документов (например, если часть ключей не проходит фильтр).

**Почему шаг FETCH нужен, если индекс уже нашёл документы:**

Индекс хранит только значения полей индекса и `_id`. Чтобы получить остальные поля документа (`route`, `status`, `error`), MongoDB обращается к коллекции по `_id` — это и есть шаг FETCH.

**Почему индекс из задачи 2 просматривает 43 ключа, а возвращает 33 документа:**

Индекс `service_1_level_1_ts_-1_duration_ms_1` находит все события с `service: "payments"` и `level: "error"` — их 43. Фильтр `duration_ms > 500` — последнее поле индекса (диапазон). MongoDB читает все 43 ключа, потом проверяет `duration_ms`, из них 33 проходят. 10 ключей — с `duration_ms <= 500`.

**Что случится с этим индексом при вставке новой записи в журнал:**

MongoDB добавит новый ключ в индекс (по значениям `service`, `level`, `ts`, `duration_ms` новой записи). Это замедлит вставку (индекс нужно обновить), но ускорит поиск.

**Почему на индексы не ставят все поля подряд:**

Каждый индекс занимает память и замедляет запись. Слишком много индексов → медленные INSERT/UPDATE, больше RAM. Индекс нужен под конкретный запрос, а не "на всякий случай".

---

## Файлы

- `plan-1.json` — запрос 1 без индекса
- `plan-2.json` — запрос 1 с индексом ESR
- `plan-3a.json` — вариант А (R, E, S)
- `plan-3b.json` — вариант Б (E, R, S)
- `plan-4-bez.json` — запрос 2 без индекса
- `plan-4.json` — запрос 2 с индексом
- `links.txt` — ссылки на разборы
