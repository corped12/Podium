# русский

# Podium 0.1.0 — фиксированный диск 8 ГиБ

Основа: тег [Leviidev/Podium 0.1.0](https://github.com/Leviidev/Podium/releases/tag/0.1.0), commit `904ffe70378a210cc7d57b2b651677c57617f1b7`.

## Что подготовлено

Новая системная файловая система и новый/стёртый Virtual iPod создаются с ёмкостью **8 589 934 592 байта (8 ГиБ)**. Это общая ёмкость тома: iOS и метаданные занимают её часть. Выбора размера нет. Образ HFS+ sparse: свободная часть не записывается нулями и не требует сразу восьми гигабайт физического места на APFS. Заполнение гостевого диска всё равно требует соответствующего свободного места на настоящем устройстве.

Исправленная IPA собрана на облачном Mac через GitHub Actions: [сборка и тесты](https://github.com/cat-yura-game/Podium/actions/runs/37121776354), исходный код commit `7c30a58`. Пройдены 546 XCTest, два пропущены, ноль ошибок. В отдельной проверке с настоящей IPSW iPod4,1 / iOS 6.1.6 новый диск 8 ГиБ загрузился до экрана блокировки: [проверка загрузки](https://github.com/cat-yura-game/Podium/actions/runs/37121774793). Проверка выполнялась на Apple Silicon в iOS Simulator с JIT. На физическом устройстве исправленная версия пока не проверялась. IPA неподписанная; её нужно подписать перед установкой.

Исправлена обнаруженная в первой сборке повторная загрузка на этапе Kernel. Прямое чтение `/dev/md0` теперь проходит через гостевую `uiomove64` и отдельную буферную страницу: адреса пользовательских процессов больше не ошибочно рассматриваются как адреса ядра. Подготовка пользовательского образа выполняется вне главного потока приложения. При раннем сбое загрузки приложение сохраняет журнал и останавливается вместо бесконечного перезапуска.

## Как устроено

- `RootFilesystemPreparer` передаёт существующему HFS+ writer фиксированную ёмкость 8 ГиБ, recipe version повышена с 13 до 14. Writer уже использует `truncate` и записи в занятые участки. Защита расчёта свободных блоков дополнена проверкой перед вычитанием размера bitmap.
- `FileBackedStorage` реализует `VirtualStorageDevice`: `pread`/`pwrite` с 64-битными смещениями, проверкой границ, повтором EINTR и обработкой ошибок хостового диска. Весь образ никогда не отображается в ARM RAM. RAM остаётся 1 ГиБ.
- `GuestDiskBridge` заменяет `mdevstrategy` и путь raw-диска на небольшие Thumb-переходники для **точного оригинального kernelcache iPod4,1 / 10B500**. XNU продолжает выполнять `buf_map`, `buf_unmap`, `buf_biodone`; raw I/O использует штатную `uiomove64` и выделяемую на каждый вызов страницу `kalloc`. Только перенос данных между образом и памятью ядра выполняет хост. Для block I/O учитываются таблицы `kernel_pmap`, включая отложенные отображения; записи в RAM обновляют кэш скомпилированного кода. Это устраняет 32-битное обрезание смещений и размеров в штатном [memdev.c](https://github.com/apple-oss-distributions/xnu/blob/xnu-2050.48.11/bsd/dev/memdev.c).
- В device tree остаётся служебный RAMDisk размером одна страница для регистрации `md0`, а ёмкость сообщается гостю через исправленные 32/64-битные capacity ioctl. Cache-sync ioctl и выключение вызывают `fsync`. Ошибка завершающего flush сохраняет возможность повторить его, как в исходном Podium.
- Проверяется SHA-256 всего распакованного ядра **до** модификации: `415717e559c48fcf8a2aedec05d3ed2efa6312b11de942acd1c56629b72192ef`. При несовпадении загрузка большого диска останавливается с понятной ошибкой. Переходники относятся только к md0; другие memory devices не поддержаны на этом пути.
- IPA/DEB и добавление файлов сохраняют исходную ёмкость при транзакционной пересборке; резерв свободного места сохраняется. Временные образы также sparse. Копирование использует APFS clone, запасной путь пропускает нулевые блоки. При временном запуске непостоянного диска записи идут в отдельную копию.

Исходные ограничения offline-установщиков остаются: поддерживаются совместимые IPA и простые DEB; установка DEB со скриптами/неподдерживаемой упаковкой, подпись и полноценная регистрация приложений в госте не добавлены. Выключение сохраняет записи, уже переданные гостем диску; это не обещание сохранения ещё не сброшенных гостевых буферов при резком Power Off.

Для сборки современным Swift-компилятором также разбито одно сложное выражение кодирования `tbl/tbx` в `A64Assembler.swift`; результат кодирования сохранён. Ручной тест загрузки локальной IPSW исключён из автоматического test target в `project.yml`: он требует отсутствующего в репозитории файла прошивки.

## Сборка и установка

На Mac с установленным Xcode и его iOS SDK:

```bash
brew install xcodegen
bash build-8gib-ipa.sh
```

Скрипт генерирует `Podium.xcodeproj` из исходного `project.yml`, собирает Release для устройства и создаёт `Podium-0.1.0-8GiB-unsigned.ipa`. Это **неподписанная** IPA. Подпиши и установи её своим инструментом sideloading, которым ты обычно устанавливаешь IPA. Либо открой сгенерированный проект в Xcode, выбери свой Team в Signing & Capabilities и установи на подключённое устройство. Исходный deployment target — iOS 17.0; инструкция оригинального релиза для JIT — StikDebug с `universal.js` на поддерживаемой версии iOS.

Для тестов:

```bash
xcodegen generate
xcodebuild -project Podium.xcodeproj -scheme Podium \
  -destination 'platform=iOS Simulator,name=YOUR_AVAILABLE_IPHONE' \
  CODE_SIGNING_ALLOWED=NO DEVELOPMENT_TEAM='' test
```

Замени `YOUR_AVAILABLE_IPHONE` именем установленного симулятора. Новые XCTest проверяют гостевые переходники в интерпретаторе и JIT, прямое чтение с невыровненных смещений, отображения памяти ядра, запись/чтение выше 4 ГиБ, последний блок, EOF и ошибки границ, изоляцию временного диска, ioctl ёмкости, переход через границы страниц и сохранение 8 ГиБ после IPA/DEB/файлов. Не загружай образ целиком через `Data(contentsOf:)` для проверки: он логически занимает 8 ГиБ.

Также приложен `.github/workflows/build-8gib.yml` для ручной сборки и тестирования на GitHub Actions после загрузки проекта в свой репозиторий. Workflow успешно выполнен в [fork пользователя](https://github.com/cat-yura-game/Podium/tree/storage-8gib). В поставку уже включена готовая `Podium-0.1.0-8GiB-unsigned.ipa`; повторная сборка для установки не обязательна. Артефакт IPA публикуется только при успешных сборке и тестах.

## Получение нового диска

1. Импортируй оригинальную IPSW **iPod4,1 / iOS 6.1.6 / 10B500**.
2. Новый Virtual iPod автоматически получит 8 ГиБ при первом запуске.
3. Уже существующий маленький `user.hfs` сохраняется вместе с приложениями и продолжает использовать прежний RAM-диск. Для перехода к 8 ГиБ сделай резервную копию нужных данных, выключи виртуальный iPod и воспользуйся штатным стиранием содержимого в Settings. **Стирание удаляет данные гостя.** Автоматическое разрушительное преобразование старого диска не выполняется.
4. Проверь в Podium ёмкость/свободное место, загрузку SpringBoard, установку совместимых IPA/DEB и сохранение данных после выключения/повторного запуска.

В архив не включены IPSW, ядро Apple или персональные сертификаты. Реальный заполненный образ создаётся из импортированной прошивки самим приложением.

## Патч и проверка переходников

На чистом теге 0.1.0:

```bash
git apply --check /path/to/Podium-0.1.0-8GiB.patch
git apply /path/to/Podium-0.1.0-8GiB.patch
```

В `StorageBridge/` сохранены исходные assembler-тексты, дизассемблирование и manifest байтов. Все гостевые адреса вызовов были сверены с экспортированными символами настоящего ядра. Проверка отдельно от Xcode:

```bash
python3 -m pip install unicorn
python3 StorageBridge/verify_shims.py
```

Можно добавить `--kernel /path/to/decompressed/kernel.macho`, чтобы проверить SHA-256 и соответствие адресов смещениям Mach-O. Эти тесты проверяют ABI переходников на подставленных XNU-функциях и **не проверяют реальную загрузку iOS или Swift-реализацию диска**. Swift-реализация проверена отдельными XCTest в успешной облачной сборке. Отдельный тест `RealDiskBootTests` проверил загрузку настоящей iOS до экрана блокировки на новом диске 8 ГиБ. Для его повторения запусти workflow `Diagnose real 8 GiB guest boot`: он временно скачивает оригинальную IPSW Apple, выполняет загрузку и сохраняет результат XCTest без публикации прошивки. Проверка на физическом устройстве остаётся необходимой.


# English

# Podium 0.1.0 — 8 GiB fixed-size disk

Base: [Leviidev/Podium 0.1.0](https://github.com/Leviidev/Podium/releases/tag/0.1.0) tag, commit `904ffe70378a210cc7d57b2b651677c57617f1b7`.

## What has been prepared

A new system filesystem and a new/wiped Virtual iPod are created with a capacity of **8,589,934,592 bytes (8 GiB)**. This is the total volume capacity; iOS and metadata occupy a portion of it. There is no option to select the size. The HFS+ image is sparse: the free space is not zero-filled and does not immediately require eight gigabytes of physical space on APFS. However, filling the guest disk still requires corresponding free space on the actual device.

The patched IPA was built on a cloud-based Mac via GitHub Actions: [build and tests](https://github.com/cat-yura-game/Podium/actions/runs/37121776354), source code commit `7c30a58`. 546 XCTests passed, two were skipped, and there were zero errors. In a separate check using a real iPod4,1 / iOS 6.1.6 IPSW, the new 8 GiB disk booted to the lock screen: [boot check](https://github.com/cat-yura-game/Podium/actions/runs/37121774793). The check was performed on Apple Silicon in the iOS Simulator with JIT enabled. The patched version has not yet been tested on a physical device. The IPA is unsigned; it must be signed before installation. Fixed a re-loading issue at the Kernel stage discovered in the first build. Direct reads from `/dev/md0` now pass through the guest's `uiomove64` and a dedicated buffer page; user-process addresses are no longer erroneously treated as kernel addresses. User image preparation is performed outside the application's main thread. In the event of an early boot failure, the application saves a log and halts instead of restarting indefinitely.

## Implementation Details

- `RootFilesystemPreparer` passes a fixed 8 GiB capacity to the existing HFS+ writer; the recipe version has been bumped from 13 to 14. The writer already utilizes `truncate` and writes to allocated sectors. The free-block calculation logic now includes a safety check before subtracting the bitmap size.
- `FileBackedStorage` implements `VirtualStorageDevice`, featuring `pread`/`pwrite` with 64-bit offsets, bounds checking, `EINTR` retries, and host disk error handling. The entire image is never mapped into ARM RAM; RAM usage remains at 1 GiB.
- `GuestDiskBridge` replaces `mdevstrategy` and the raw disk path with small Thumb-mode shims to ensure compatibility with the **exact original iPod4,1 / 10B500 kernelcache**. XNU continues to execute `buf_map`, `buf_unmap`, and `buf_biodone`; raw I/O utilizes the standard `uiomove64` and a `kalloc` page allocated per call. The host handles only the data transfer between the image and kernel memory. Block I/O operations account for `kernel_pmap` tables, including deferred mappings; writes to RAM update the compiled code cache. This eliminates the 32-bit truncation of offsets and sizes found in the standard [memdev.c](https://github.com/apple-oss-distributions/xnu/blob/xnu-2050.48.11/bsd/dev/memdev.c).
- A single-page RAMDisk remains in the device tree to register `md0`, while the actual capacity is reported to the guest via patched 32/64-bit capacity ioctls. Cache-sync ioctls and shutdown operations trigger `fsync`. If the final flush fails, the operation can be retried, just as in the original Podium.
- The SHA-256 hash of the entire unpacked kernel is verified **before** modification: `415717e559c48fcf8a2aedec05d3ed2efa6312b11de942acd1c56629b72192ef`. If the hash does not match, the large disk loading process halts with a clear error message. These adaptations apply only to `md0`; other memory devices are not supported in this workflow.
- IPA/DEB processing and file additions preserve the original capacity during transactional rebuilding; the free space reserve is maintained. Temporary images are also created as sparse files. Copying utilizes APFS cloning, while the fallback path skips zero blocks. When launching a non-persistent disk temporarily, writes are directed to a separate copy.

The original limitations of offline installers remain: support is limited to compatible IPAs and simple DEBs; features such as installing DEBs with scripts or unsupported packaging formats, signing, and full application registration within the guest have not been added. Shutdown preserves writes already committed by the guest to the disk; this does not guarantee the preservation of guest buffers that have not yet been flushed in the event of an abrupt power-off. To ensure compatibility with the modern Swift compiler, a complex `tbl/tbx` encoding expression in `A64Assembler.swift` has been broken down, while preserving the encoding result. The manual test for loading a local IPSW has been excluded from the automated test target in `project.yml`, as it requires a firmware file not present in the repository.

## Build and Installation

On a Mac with Xcode and the iOS SDK installed:

```bash
brew install xcodegen
bash build-8gib-ipa.sh
```

The script generates `Podium.xcodeproj` from the source `project.yml`, builds the Release version for the device, and creates `Podium-0.1.0-8GiB-unsigned.ipa`. This is an **unsigned** IPA. Sign and install it using your preferred sideloading tool. Alternatively, open the generated project in Xcode, select your Team under "Signing & Capabilities," and install it on a connected device. The initial deployment target is iOS 17.0.



Установи исправленную IPA поверх предыдущей с тем же идентификатором приложения и способом подписи, чтобы сохранить данные Podium. После установки снова включи JIT и запусти виртуальный iPod. Для уже созданного диска 8 ГиБ повторное стирание не требуется: исправление находится в эмуляторе. Сохрани резервную копию перед обновлением; удаление приложения может удалить импортированную прошивку и диск.
