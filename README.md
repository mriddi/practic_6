# Практическая работа 6 (вариант на Python). Аудит репозитория

## Цель

Провести аудит безопасности репозитория тремя классами инструментов — анализ кода (Semgrep), анализ состава ПО (Syft и Grype), поиск секретов (Gitleaks), — расставить приоритеты устранения по таблице решений и предложить, как мониторинг будет наблюдать за тем, что нельзя закрыть сразу.

Практика курса «Мониторинг компьютерных атак». Вместо больших учебных проектов — три маленьких веб-сервиса на Python, написанных специально для практики. Каждый — 2–5 файлов кода, его можно прочитать целиком за несколько минут, и каждая находка видна прямо в коде.

## Продолжительность и порядок

2 часа (120 минут). Сначала все вместе с преподавателем выполняют **вариант A**, команды — на [слайдах практики](slides/slides-lab-06py.pdf). Затем каждый самостоятельно выполняет **вариант B** или **вариант C** и сдаёт результаты по нему.

| Вариант | Репозиторий | Что это | Чем отличается |
|---|---|---|---|
| A (вместе) | `library.bundle` | Каталог библиотеки на Flask | В `requirements.txt` закреплены только прямые зависимости |
| B | `shop.bundle` | Интернет-магазин на Flask | Закреплены все версии; есть `requirements-dev.txt` с пакетами для разработки |
| C | `clinic.bundle` | Запись к врачу на Django | Часть зависимостей без версий |

Репозитории лежат в каталоге [`repos/`](repos/) как git-bundle — один файл со всей историей коммитов. Файл клонируется так же, как репозиторий с GitHub, и результаты не зависят от дня выполнения. «Секреты» в репозиториях сгенерированы случайно и нигде не действуют.

## Шаг 0. Контейнер и инструменты

Учебные репозитории намеренно уязвимы и содержат «секреты», поэтому вся работа идёт в контейнере: одинаковая среда у всех, ничего не попадает в домашний каталог, после занятия контейнер удаляется одной командой. Приложения не запускаются.

### 0.1. Запуск контейнера (на своём компьютере)

Нужен Docker: Docker Desktop для Windows и macOS или Docker Engine для Linux. Проверка: `docker version` выводит версии клиента и сервера.

```bash
mkdir -p lab06/work && cd lab06
curl -sSfL -o work/library.bundle https://github.com/mriddi/practic_6/raw/main/repos/library.bundle
docker run -it --name audit -v "$PWD/work:/work" -w /work python:3.11-bookworm bash
```

- Для варианта B скачайте `shop.bundle`, для C — `clinic.bundle` (тем же адресом, с другим именем файла).
- В Windows PowerShell: `-v "${PWD}\work:/work"`.
- Каталог `work/` на компьютере виден в контейнере как `/work`: клон и все отчёты остаются на компьютере.
- Вернуться в контейнер после `exit` — `docker start -ai audit`; второй терминал — `docker exec -it audit bash`.
- Для варианта B или C после варианта A — новый каталог (`mkdir -p lab06-b/work && cd lab06-b`) и новый контейнер (сначала `docker rm audit` или другое имя в `--name`).

Образ `python:3.11-bookworm` (Debian 12, Python 3.11, git) собран для `amd64` и `arm64` — подходит и для Mac на Apple Silicon.

**Все команды ниже выполняются внутри контейнера, в каталоге `/work`.**

### 0.2. Проверка среды и Semgrep

```bash
cat /etc/debian_version
uname -m
python3 --version; pip --version; git --version

python3 -m venv /opt/semgrep
/opt/semgrep/bin/pip install -q "semgrep==1.136.0"
ln -s /opt/semgrep/bin/semgrep /usr/local/bin/semgrep
semgrep --version
```

Semgrep ставится в отдельное виртуальное окружение, чтобы его зависимости не смешались с зависимостями проверяемого проекта.

### 0.3. Syft, Grype, Gitleaks: закреплённые версии и контрольные суммы

```bash
ARCH=$(dpkg --print-architecture); echo $ARCH
mkdir -p /tmp/dl && cd /tmp/dl
V=1.54.1; U=https://github.com/anchore/syft/releases/download/v$V
curl -sSfLO $U/syft_${V}_linux_${ARCH}.tar.gz
curl -sSfLO $U/syft_${V}_checksums.txt
sha256sum --check --ignore-missing syft_${V}_checksums.txt
tar xzf syft_${V}_linux_${ARCH}.tar.gz syft && install syft /usr/local/bin/
V=0.120.1; U=https://github.com/anchore/grype/releases/download/v$V
curl -sSfLO $U/grype_${V}_linux_${ARCH}.tar.gz
curl -sSfLO $U/grype_${V}_checksums.txt
sha256sum --check --ignore-missing grype_${V}_checksums.txt
tar xzf grype_${V}_linux_${ARCH}.tar.gz grype && install grype /usr/local/bin/
V=8.30.1; U=https://github.com/gitleaks/gitleaks/releases/download/v$V
GA=$( [ $ARCH = amd64 ] && echo x64 || echo $ARCH )
curl -sSfLO $U/gitleaks_${V}_linux_${GA}.tar.gz
curl -sSfLO $U/gitleaks_${V}_checksums.txt
sha256sum --check --ignore-missing gitleaks_${V}_checksums.txt
tar xzf gitleaks_${V}_linux_${GA}.tar.gz gitleaks && install gitleaks /usr/local/bin/
cd /work
syft version | head -2
grype version | head -2
gitleaks version
```

Каждая проверка `sha256sum` должна вывести `…: OK`.

### 0.4. Настройка и целевой репозиторий

Отключите проверку новых версий Syft и Grype (в контейнере без IPv6 она ждёт 10 секунд и печатает ошибку):

```bash
echo 'check-for-app-update: false' > ~/.syft.yaml
echo 'check-for-app-update: false' > ~/.grype.yaml
```

Клонируйте репозиторий своего варианта (`library`, `shop` или `clinic`) с полной историей — Gitleaks ищет секреты во всех коммитах:

```bash
ls -l *.bundle
git clone -q library.bundle target-repo
git -C target-repo log --format="%h %ad %s" --date=short
mkdir -p results
ls -a target-repo
```

Все результаты складывайте в `results/`.

Проверено на версиях: Semgrep 1.136.0, Syft 1.54.1, Grype 0.120.1, Gitleaks 8.30.1, база Grype от 23.10.2026.

> **Ошибка `network is unreachable` или `i/o timeout`.** Если Syft, Grype или pip выдают её при загрузке, это временный сбой сети в контейнере. Повторите команду.

## Задание

### Шаг 1. Анализ кода — Semgrep (25 мин)

Отчёты читаются табличным выводом самих инструментов и стандартными `grep`, `sort`, `uniq -c`, `wc -l`. JSON-отчёты сохраняются для сдачи.

1. Запустите Semgrep с явно указанными наборами правил (`--config auto` не используйте: он требует отправки метрик на серверы Semgrep):

   ```bash
   semgrep scan --config p/default --metrics=off --json-output=results/semgrep.json --text-output=results/semgrep.txt target-repo/
   semgrep scan --config p/owasp-top-ten --metrics=off --json-output=results/semgrep-owasp.json target-repo/
   ```

2. Посчитайте находки по уровню, выпишите самые частые номера CWE и правила, найдите места, где на одной строке сработало несколько правил:

   ```bash
   grep -o '"severity":"[A-Z]*"' results/semgrep.json | sort | uniq -c
   grep -o '"CWE-[0-9]*' results/semgrep.json | sort | uniq -c | sort -rn | head -5
   grep -o '"check_id":"[^"]*"' results/semgrep.json | sort | uniq -c | sort -rn | head -5
   grep -o '"check_id":"[^"]*","path":"[^"]*","start":{"line":[0-9]*' results/semgrep.json | sed 's/"check_id":"[^"]*",//' | sort | uniq -c | awk '$1 > 1'
   ```

3. Выведите находки уровня ERROR — с правилом, сообщением и фрагментом кода (`❯❯❱` — ERROR, `❯❱` — WARNING):

   ```bash
   semgrep scan --config p/default --metrics=off --severity ERROR target-repo/
   ```

   Код вокруг находки смотрите сами: `sed -n '<с>,<по>p' <файл>`. Файлы маленькие — их можно открыть целиком: `cat target-repo/<файл>`.

4. Выберите пять находок уровня ERROR и для каждой запишите: фрагмент кода, CWE, в чём уязвимость, как исправить.
5. Проверьте отчёт на три проблемы, разобранные в лекции:
   - *одна ошибка — несколько находок*: сколько *уязвимостей* стоит за всеми находками ERROR;
   - *неточные метаданные*: есть ли находки, у которых CWE или уровень не соответствует сути;
   - *неподходящие рекомендации*: есть ли советы для другой библиотеки или фреймворка.
6. Отметьте хотя бы одно ложное или спорное срабатывание. Прочитайте код целиком: какую уязвимость Semgrep *не нашёл*?

### Шаг 2. Состав ПО — Syft и Grype (25 мин)

7. Постройте SBOM по репозиторию и выясните, *из чего* он построен (источник пакета есть только в JSON, `syft:location:0:path`):

   ```bash
   cat target-repo/requirements*.txt
   syft dir:target-repo -o table -o cyclonedx-json=results/sbom.json 2>/dev/null > results/sbom.txt
   cat results/sbom.txt
   grep -o '"syft:location:0:path","value":"[^"]*"' results/sbom.json | sort | uniq -c
   ```

   Все ли строки файла зависимостей попали в SBOM? Есть ли в нём транзитивные зависимости (те, что нужны самим библиотекам)?

8. Проверьте SBOM на уязвимости. Дату базы запишите. `--by-cve` выводит номера CVE; таблица отсортирована по оценке RISK, уязвимости из каталога KEV помечены `(kev)`:

   ```bash
   grype db update
   grype db status
   grype sbom:results/sbom.json --by-cve -o table -o json=results/grype.json > results/grype.txt
   head -11 results/grype.txt
   tail -n +2 results/grype.txt | wc -l
   grep -oE ' (Critical|High|Medium|Low|Negligible|Unknown) ' results/grype.txt | sort | uniq -c | sort -rn
   awk 'NR > 1 {print $1, $2}' results/grype.txt | sort | uniq -c | sort -rn
   grep kev results/grype.txt
   ```

   `grype db status` должен показать `Status: valid`; без базы Grype создаёт пустой `grype.json`.

9. Постройте второй SBOM — по тому, что устанавливается на самом деле. `pip install --target libs` ставит зависимости вместе с транзитивными в отдельный каталог `libs`, ничего не запуская; `diff` показывает разницу двух SBOM (`<` — только в первом, `>` — только во втором):

   ```bash
   pip install -q --disable-pip-version-check --root-user-action=ignore --target libs -r target-repo/requirements.txt
   ls libs
   syft dir:libs -o table -o cyclonedx-json=results/sbom-installed.json 2>/dev/null > results/sbom-installed.txt
   cat results/sbom-installed.txt
   diff <(awk 'NR > 1 {print $1, $2}' results/sbom.txt) <(awk 'NR > 1 {print $1, $2}' results/sbom-installed.txt)
   grype sbom:results/sbom-installed.json --by-cve -o table -o json=results/grype-installed.json > results/grype-installed.txt
   tail -n +2 results/grype-installed.txt | wc -l
   awk 'NR > 1 {print $1, $2}' results/grype-installed.txt | sort | uniq -c | sort -rn
   ```

   Каких пакетов не было в первом SBOM и почему? Какие версии получили незакреплённые пакеты и от чего это зависит?

10. Через что пришёл уязвимый пакет: у каждого установленного пакета в `libs/<пакет>.dist-info/METADATA` перечислены его зависимости (строки `Requires-Dist`):

    ```bash
    grep -l "^Requires-Dist: urllib3" libs/*.dist-info/METADATA
    ```

11. Для пяти уязвимостей с наибольшей оценкой RISK заполните таблицу. CVSS и оценки CISA — на странице CVE в NVD: `https://nvd.nist.gov/vuln/detail/<CVE>`, блок *CISA ADP Vulnrichment* (SSVC: Exploitation, Automatable, Technical Impact). Наличие в KEV — пометка `(kev)` в таблице Grype или поиск в каталоге https://www.cisa.gov/known-exploited-vulnerabilities-catalog.

    | CVE | Пакет | Версия | Исправлено в | CVSS | EPSS (вероятность и процентиль) | В KEV? | Automatable / Technical Impact |
    |---|---|---|---|---|---|---|---|
    | | | | | | | | |

### Шаг 3. Секреты — Gitleaks (10 мин)

12. Проверьте и текущие файлы, и историю. `-v` печатает каждую находку подробно, `--redact` скрывает значения секретов:

    ```bash
    gitleaks dir target-repo -v --redact --report-format json --report-path results/gitleaks-dir.json > results/gitleaks-dir.txt 2>&1
    tail -1 results/gitleaks-dir.txt
    grep -E '^(RuleID|File|Line):' results/gitleaks-dir.txt
    gitleaks git target-repo -v --redact --report-format json --report-path results/gitleaks.json > results/gitleaks-git.txt 2>&1
    tail -1 results/gitleaks-git.txt
    grep -E '^(RuleID|File|Commit|Date):' results/gitleaks-git.txt
    grep '^File' results/gitleaks-git.txt | awk '{print $2}' | sort -u | while read f; do [ -e "target-repo/$f" ] || echo "$f"; done
    ```

    Последняя команда выводит файлы с находками, которых больше нет в репозитории. Посмотреть файл в старом коммите: `git -C target-repo show <коммит>:<файл>`.

13. Какие секреты есть только в истории? Для каждой находки запишите тип секрета, файл, коммит, правило Gitleaks и признак — реальный это секрет, тестовый или ложное срабатывание. Какие секреты в коде Gitleaks *не нашёл*?
14. Для реальных секретов опишите действия: отзыв и выпуск нового, проверка использования (см. шаг 5). Удаление файла из репозитория к действиям не относится — объясните почему.

### Шаг 4. План устранения (25 мин)

15. Для каждой находки ответьте на четыре вопроса таблицы BOD 26-04 из лекции и определите срок. Считайте, что приложение доступно из интернета. Значения «автоматизируема» и «полный/частичный контроль» берите из оценок CISA (п. 11), а где их нет — оцените сами и обоснуйте.
16. Для двух-трёх уязвимостей зависимостей проверьте *достижимость*: подключается ли пакет и вызывается ли уязвимая функция в коде проекта. Все подключения библиотек в коде:

    ```bash
    grep -rn "^import\|^from" target-repo --include=*.py
    ```

    Если уязвимый код не вызывается — оформите вывод «не затронут» с причиной (как запись VEX).
17. Составьте единый план устранения (не меньше 10 строк):

| № | Находка | Инструмент | CWE / CVE | Интернет / KEV / Автом. / Контроль | Срок | Действие | Владелец | Проверка |
|---|---|---|---|---|---|---|---|---|
| 1 | Ключ AWS в истории `.env` | Gitleaks | — | вне таблицы | немедленно | Отозвать ключ, проверить журнал аудита | DevOps | Ключ не действует; отчёт по журналу |
| 2 | SQL-инъекция в поиске | Semgrep | CWE-89 | Да / — / Да / Частичный | 14 дней | Параметризованный запрос | Разработка | Semgrep молчит; тест на `' OR '1'='1` |

18. Сравните свой порядок с порядком по CVSS: какие находки поднялись, какие опустились и почему?
19. Для одной находки, которую нельзя исправить в срок, оформите исключение: причина, компенсирующая мера, владелец, срок действия.
20. Предложите проверки для конвейера CI/CD: на каком этапе запускается каждый инструмент, что блокирует слияние, что только предупреждает. Используйте приёмы лекции: только новые находки, только исправимое, порог по сроку из таблицы.

### Шаг 5. Мониторинговый мост (15 мин)

Опирайтесь на раздел лекции «Мониторинг, пока уязвимость не закрыта».

21. Выберите три находки, которые по плану закрываются не сразу, и для каждой опишите компенсирующее правило:
    - *что ищем* — признак эксплуатации именно этой уязвимости (адрес и параметр запроса, процесс, соединение);
    - *в каком журнале* — и попадает ли туда нужное поле сейчас (если нет — это требование к журналированию);
    - *как отличить попытку от успеха* — какое второе событие подтверждает, что атака удалась;
    - *ожидаемые ложные срабатывания*.
22. Для найденного секрета опишите проверку его использования: какой журнал, какой фильтр, за какой период, что считать признаком злоупотребления. Хватит ли срока хранения журнала, чтобы проверить весь период с первого коммита?
23. Добавьте в каталог событий две строки для событий цепочки поставок вашего проекта: тип события, источник, что показывает, назначение (сигнал, контекст или задача).

## Критерии оценки

- [ ] Semgrep: пять находок разобраны; найдены повторы, неточные метаданные или неподходящие рекомендации; отмечено ложное срабатывание и пропуск
- [ ] SBOM: определено, из чего он построен; есть второй SBOM и сравнение двух
- [ ] Grype: таблица пяти уязвимостей с EPSS и KEV
- [ ] Gitleaks: сравнение текущих файлов и истории; секреты классифицированы; действия по реальным секретам
- [ ] План устранения: сроки по таблице BOD 26-04, проверка достижимости, сравнение с порядком по CVSS, одно оформленное исключение
- [ ] Проверки для CI/CD с явными порогами блокировки
- [ ] Три компенсирующих правила, проверка использования секрета, две строки каталога событий

## Результаты

1. `results/semgrep.json`, `results/sbom.json`, `results/grype.json`, `results/sbom-installed.json`, `results/grype-installed.json`, `results/gitleaks-dir.json`, `results/gitleaks.json`
2. `results/plan.md` — разбор находок, план устранения, исключение, проверки для CI/CD
3. `results/monitoring.md` — компенсирующие правила, проверка секрета, строки каталога

После занятия контейнер можно удалить: `docker rm -f audit` (на компьютере, не в контейнере). Результаты останутся в `work/results/`.

## Проверка перед сдачей

```bash
ls -l results/
```

Пустой `grype.json` означает, что не скачалась база уязвимостей: повторите `grype db update`.
