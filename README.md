# Home Assistant + Matter Server на Ubuntu

Два контейнери з `network_mode: host`, розгортання через Coolify з GitHub.
Matter Server використовує matter.js. Home Assistant зберігає детальну історію
90 днів; довгострокова погодинна статистика підтримуваних датчиків зберігається
окремо без автоматичного видалення.

## Перед розгортанням

- Потрібен Ubuntu з Docker Engine і Docker Compose v2+.
- Ubuntu, розетка й iPhone для додавання пристрою мають бути в одній домашній
  LAN/VLAN. Ubuntu може бути підключений Ethernet, розетка — Wi-Fi.
- На LAN-інтерфейсі потрібен IPv6, а в мережі — проходження multicast/mDNS
  без ізоляції Wi-Fi-клієнтів. IPv6 від інтернет-провайдера не потрібен.
- Порти хоста 8123 та 5580 мають бути вільними.
- Стек призначений для Linux. На Mac його тут не запускаємо.

## Coolify

1. Запуш репозиторій на GitHub.
2. У Coolify створи **Application** із цього GitHub-репозиторію, гілка `main`.
3. Обери **Docker Compose** як build strategy, Base Directory `/`,
   Compose Location `/compose.yaml` (або `compose.yaml`, залежно від UI).
4. Вибери Ubuntu-сервер у домашній мережі як destination.
5. У **Advanced → Docker compose → Compose deployment** обери
   **Raw (deploy file as-is)**. Не підключай сервіси до predefined network:
   обидва вже використовують мережу Ubuntu.
6. У **Environment Variables** задай обов'язкову змінну `LAN_INTERFACE`.
   Приклад: `enp3s0`. Реальна назва визначається на Ubuntu:

   ```sh
   ip -br addr
   ip -6 addr show dev enp3s0
   ```

   Решта змінних необов'язкові й описані в `.env.example`. Переконайся,
   що змінні доступні під час розгортання Compose, а не лише build.
7. Вимкни preview deployments для цього Application: host-порти й volumes
   призначені для одного постійного екземпляра на сервері.
8. Залиш **Domains** порожнім і натисни **Deploy**.
9. Відкрий `http://<LAN-IP-Ubuntu>:8123` і створи обліковий запис HA.

Raw-режим не створює автоматичний маршрут через Coolify proxy. Для початкового
доступу використовуємо LAN-IP. Якщо пізніше потрібен домен із HTTPS, окремо
налаштуємо reverse proxy та довірені проксі Home Assistant.

Стек не змінює firewall Ubuntu. Він має дозволяти доступ до 8123 із домашньої
мережі та потрібний Matter-трафік на LAN-інтерфейсі. Порти не потрібно
переадресовувати в інтернет. API Matter слухає лише `127.0.0.1:5580`.

## Додавання розетки, яка вже в Apple Home

1. У HA відкрий **Settings → Devices & services → Add integration → Matter**.
2. Якщо пропонується керований Matter Server app, зніми відповідну галочку.
3. Вкажи адресу зовнішнього сервера:

   ```text
   ws://127.0.0.1:5580/ws
   ```

   Саме `127.0.0.1`: HA і Matter Server використовують одну мережу Ubuntu.
   Назва сервісу `matter-server` тут не використовується як Docker DNS.
4. Встанови Home Assistant Companion на iPhone і підключи його до цього HA.
5. У Companion відкрий **Settings → Connectivity → Matter → Add device →
   Yes, it's already in use → Apple Home** та виконай інструкції спільного доступу.
6. Перевір датчики розетки у HA. Apple Home працюватиме паралельно.

Для цього сценарію використовується телефон; Bluetooth-адаптер Ubuntu,
`privileged: true`, D-Bus і Thread Border Router для Wi-Fi-розетки не потрібні.

## Потужність та споживання

- Датчик **Вт**: додай на dashboard і переглядай його History.
- Датчик **кВт·год**: додай як individual device у налаштуваннях Energy.
- Якщо є лише Вт: створи **Integral / Integration** helper з виходом у кВт·год.
  Це розрахункова енергія; точність залежить від частоти вимірювань і пропусків.
- Для лічильників за годину/день/місяць створи **Utility Meter** helpers
  із датчика кВт·год.

Доступність датчиків Вт/кВт·год залежить від прошивки розетки та того, які
Matter-атрибути вона передає. Потрібно перевірити після спарювання. Частота
оновлення визначається пристроєм; збереження історії не робить її щосекундною.
За час до підключення HA історія автоматично не з'явиться.

Конфігурація HA зберігається безпосередньо в `compose.yaml`, у стартовій команді
сервісу `homeassistant`. Перед запуском штатного `/init` вона записується
атомарною заміною в `/config/configuration.yaml` усередині named volume.
Щоб змінити строк історії, відредагуй `purge_keep_days` у Compose, закоміть і redeploy.
Ручні зміни саме `/config/configuration.yaml` будуть перезаписані при наступному
старті контейнера. Файли з checkout репозиторію не монтуються: це дозволяє
розгортати стек через helper-контейнер Coolify без шляхів `/artifacts/...` на Ubuntu.
Облікові записи, інтеграції, helpers, dashboard та база даних залишаються
у записуваному volume `/config`.

## Дані й резервні копії

| Volume за замовчуванням | Вміст |
| --- | --- |
| `homeassistant_config` | Облікові записи, інтеграції, SQLite, історія, dashboards |
| `homeassistant_matter` | Matter fabric, ключі та спарені пристрої |

Docker створює named volumes автоматично; новий Matter volume отримує права
UID/GID 1000 із директорії `/data` офіційного образу. Ручний `chown` не потрібний.
Це стосується нового volume; для перенесення наявних даних права перевіряються окремо.

Звичайний redeploy або перезапуск не видаляє volumes. Не змінюй `STACK_ID`
після першого запуску: інше значення вибере інші volumes і виглядатиме як нова
інсталяція. Не виконуй `docker compose down -v` і не видаляй volumes у Coolify.

Для резервної копії зупини Application у Coolify, експортуй **обидва** volumes
засобами резервування Docker volumes і знову запусти Application. Збережи також
цей репозиторій та значення `STACK_ID`. Зберігай копію поза Ubuntu-сервером:
дані Matter містять ключі, а HA — облікові дані інтеграцій. Для відновлення
потрібні обидва volumes; Git сам по собі не містить історії або спарювання.

## Оновлення та перевірка

Якщо попередній deploy завершився помилкою `bind source path does not exist:
/artifacts/.../homeassistant/configuration.yaml`, запуш виправлений Compose,
перезавантаж його визначення в Coolify та повтори deploy з нового коміту.
`LAN_INTERFACE=enp58s0` залиш без змін. Видаляти volumes для цього виправлення
не потрібно. Назви й призначення volumes у Compose не змінені; у managed-режимі
Coolify може додавати до їхніх фактичних назв UUID ресурсу.

`HA_VERSION` і `MATTER_VERSION` спочатку мають значення `stable`. Зафіксуй
перевірені теги версій перед регулярним використанням. Перед оновленням зроби
резервну копію й явно перевір, що Coolify підтягнув нові образи. Відкат Git
не відкочує зміни формату даних, зроблені новою версією застосунку.

У Coolify перевір Runtime Logs та здоров'я обох сервісів. Після налаштування
перевір надходження нових показів і запис History, а також збереження
інтеграцій та історії після redeploy. Matter використовує health check образу;
HA має HTTP health check у Compose.

Локальна перевірка синтаксису без запуску контейнерів:

```sh
LAN_INTERFACE=enp3s0 docker compose config --quiet
```

Альтернативний запуск через CLI **на Ubuntu**:

```sh
cp .env.example .env
# Відредагуй LAN_INTERFACE у .env.
docker compose config --quiet
docker compose up -d
docker compose ps
docker compose logs --tail=100 homeassistant matter-server
```

## Документація

- [Home Assistant Container на Linux](https://www.home-assistant.io/installation/linux/)
- [Інтеграція Matter та спільний доступ з Apple Home](https://www.home-assistant.io/integrations/matter/)
- [Matter Server: Docker та мережеві налаштування](https://github.com/matter-js/matterjs-server/blob/main/docs/docker.md)
- [Coolify: Git-based Docker Compose та Raw mode](https://coolify.io/docs/applications/builds/docker-compose)
- [History та довгострокова статистика](https://www.home-assistant.io/integrations/history/)

Home Assistant OS із Matter Server app є офіційно рекомендованим шляхом для
Matter. У цьому репозиторії використовується самостійно обслуговувана
контейнерна конфігурація відповідно до обраного способу розгортання.
