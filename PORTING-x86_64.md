# План работ: CactZstd x86_64 (CactOS 3.0.0)

Порт CLI-утилиты **zstd** на CactOS.  Репозиторий — **форк** upstream zstd:
дерево в корне и есть upstream (пин — тег v1.5.7), а вся Cact-часть — коммиты
ветки `cact`, поставленные поверх тега; различия окружения выражаются флагами
сборки.

## Зачем

`firmextr` (самодельный распаковщик ustar+zstd) убирается: загрузочные модули
(`firmrepo`, `netdrvrepo`, `drvrepo`, `localbinrepo`) — это ustar-образ в одном
zstd-фрейме, и его вскрывает обычная связка `zstd -dc <архив> | tar -xf -`.
Установщик и шелл используют ту же утилиту.

## Решения

1. **Форк upstream-репозитория** (как CactLlvm и остальные порты), а не
   приложение в CactUserBins: upstream-дерево большое, у него свой цикл
   синхронизации, и форк несёт историю upstream плюс один коммит-порт.
2. **Один статический линк** — программа `zstd` (PIE @ 0x08000000, `link.ld`,
   интерпретатор `/lib/ld.so`), собранная из `lib/*` и `programs/*` против
   CactLibc.  `start.o`/`clibc.so` берутся из `-Dcactlib`.
3. **Многопоточная сборка.** `ZSTD_MULTITHREAD` включается: pthreads CactLibc
   поддерживают `threading.c`, пул воркеров и `fileio_asyncio.c`, так что `-T#`
   работает.  Всё остальное (сжатие, распаковка, `-l`, `--train`, бенч `-b`,
   legacy v0.5+) тоже включено.
4. **Ничего в upstream-исходниках не правится.**  Различия окружения
   выражаются флагами: `-D__CACT__`, `-DPLATFORM_POSIX_VERSION=200809L`
   (иначе `programs/platform.h` не находит `_POSIX_VERSION` и выключает
   POSIX-пути), `-DZSTD_LEGACY_SUPPORT=5`, `-DZSTD_MULTITHREAD`,
   `-DZSTD_DISABLE_ASM`.

## Этапы

### Фаза 0 — репозиторий и сборка

**Milestone:** `ninja -C build-meson` линкует `build-meson/zstd`;
`ninja -C build-meson stage` кладёт его в `-Dlr_bin`.

### Фаза 1 — замена firmextr и установщик

**Milestone:** `firmextr` убран из `CactUserBins`; `cact-install` читает архивы
через `zstd -dc | tar`; boot-документация ссылается на `zstd`.

## Проверки

- **Сборка** — `ninja -C build-meson`.
- **ELF** — `readelf -h`: `ET_EXEC` @ 0x08000000, интерпретатор `/lib/ld.so`,
  `NEEDED clibc.so` (как у остальных утилит CactUserBins).

## Открытые вопросы

- **asm-распаковка (BMI2)** — выключена (`-DZSTD_DISABLE_ASM`).  Сам
  `huf_decompress_amd64.S` clang'ом ассемблируется (проверено), но zstd включает
  asm только под `__linux__`/`__APPLE__`/Windows (`portability_macros.h`), а
  объявлять `__linux__` всему фронтенду ради ускорения только распаковки — это
  смена платформы для `platform.h`/`util.c` (кадр trace, подсчёт ядер).  Решение:
  держим выключенной; включать осознанно, если понадобится скорость.
- **Эталонный вывод в гостя** — проверить `zstd -dc`/`-z`/`-T#` в CactOS
  (хост-сборка проверена, гостевого прогона нет).
