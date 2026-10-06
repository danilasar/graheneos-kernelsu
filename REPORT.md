# Отчёт: экспериментальная интеграция KernelSU Next на Pixel 8 / GrapheneOS

**Обновлено:** 2026-10-07  
**Устройство:** Google Pixel 8 (`shiba`)  
**Проверенная система:** GrapheneOS `2026100201`, Android 17  
**Архив документов/артефактов:** `ksu-next-shiba-2026100201-release/`

> Это хронологический отчёт; ранние разделы фиксируют неудачные промежуточные состояния. Их заменяют итоговые разделы 17–23 и отдельные INSTALL/UPDATE-инструкции. Исходные kernel repositories и временные build/bugreport logs впоследствии удалены по запросу пользователя; сохранены отчёт, инструкции, patch set и release artifacts.

## Итоговое состояние

- KernelSU Next v3.4.0 встроен в kernel image (`CONFIG_KSU=y`). Исправление чтения user pointers вернуло zygote/manager scan, а `prctl` transport обходит GrapheneOS seccomp block для `reboot(2)`.
- Проверенный 64-MiB образ с AVB security patch property `2026-09-05` записан **только в `boot_b`**. `init_boot_b` ранее менялся отдельным экспериментом; последние установки его не трогали. `vendor_kernel_boot`, `vendor_dlkm`, `system_dlkm`, `dtbo` и `vbmeta` не прошивались на заключительном этапе.
- Телефон загрузился на слоте `_b`; проверка `adb shell su -c id` вернула UID 0 в контексте `u:r:ksu:s0`. Termius/ADB root use works via `su`; `adb root` — отдельная функция и не означает root для production `adbd`.
- Neo Backup (`com.machiav3lli.backup`) заработал после добавления capabilities в его отдельный KernelSU profile. На текущем телефоне профиль показывает все capabilities данной сборки; минимальный набор не установлен.
- Финальный образ проверен на GrapheneOS build `2026100201` (`CP3A.260905.009`). Не использовать на другом build без пересборки/проверки.
- На финальном этапе `fastboot -w` не запускался. Разблокировка загрузчика и первоначальные эксперименты описаны в истории, но текущая install/update-инструкция рассчитана на устройство без потери данных кроме необходимого unlock wipe.

## 1. Состояние устройства до эксперимента

По проверкам ADB устройство было на официальной GrapheneOS:

```text
ro.build.version.incremental = 2026100201
ro.build.fingerprint = google/shiba/shiba:17/CP3A.260905.009/2026100201:user/release-keys
ro.boot.slot_suffix = _b
uname -r = 6.1.177-android14-11-gf6f62ba75ac9
ro.boot.vbmeta.device_state = unlocked
ro.boot.verifiedbootstate = orange
```

Основная команда чтения состояния:

```bash
adb shell 'getprop ro.build.version.incremental; \
  getprop ro.build.fingerprint; \
  getprop ro.boot.slot_suffix; \
  uname -r; \
  getprop ro.boot.vbmeta.device_state; \
  getprop ro.boot.verifiedbootstate'
```

Текущий слот B и разблокированный загрузчик позднее подтвердили командами fastboot. Физический размер `boot_b` — `0x4000000` (64 МиБ).

## 2. Исходники и сборка

### Kernel ACK и KernelSU Next

Репозиторий ядра находится в:

```text
android-kernel-build/kernel_pixel_6.1
```

ACK `common/ack` на коммите `f6f62ba75ac9dbcf802b2ca2ebce2058362a04d6`, соответствующем версии ядра на телефоне. В репозитории доступен тег `2026100200`; тега `2026100201` там нет. Поэтому точное совпадение исходников с полным релизом GrapheneOS `2026100201` не утверждается.

KernelSU Next:

```text
version/tag: 3.4.0 / v3.4.0
source commit: 1a879d6a866f80b1fa1c1009a2ffa747873cbb5e
build version reported by Kbuild: 33294
```

В build-логе видно, что KernelSU Next собирался как часть kernel target:

```text
-- KernelSU-Next version: 33294
-- KernelSU-Next tag: v3.4.0
INFO: Build completed successfully, 84 total actions
```

### Основная команда сборки

Финальная успешная сборка выполнялась из каталога kernel repository:

```bash
cd android-kernel-build/kernel_pixel_6.1
KLEAF_REPO_MANIFEST=aosp_manifest.xml ./build_shusky.sh --lto=full
```

Лог финальной сборки:

```text
kernel_pixel_6.1/build-ksu-vermagic-final.log
```

Сохранились и предыдущие логи попыток сборки (`build-stock*.log`, `build-ksu*.log`). В ходе работы обходились проблемы локальной среды/сборочных правил Kleaf, DDK UAPI headers и `sign-file`; системные пакеты не устанавливались.

### Существенные локальные изменения сборки

В рабочем дереве остались экспериментальные изменения:

- `build/kernel/_setup_env.sh` — в Bazel sandbox явно передаются версия и тег KernelSU Next, поскольку sandbox не содержит вложенные `.git`-метаданные:
  ```bash
  KSU_GIT_VERSION=3294 KSU_GIT_TAG=v3.4.0 KSU_GIT_VERSION_VALID=1
  ```
- `build/kernel/kleaf/impl/stamp.bzl` — stable SCM version жёстко задан как `-gf6f62ba75ac9`, чтобы итоговая версия модулей совпала с установленной GrapheneOS. Это локальный экспериментальный override, а не универсальное изменение Kleaf.
- `private/devices/google/common/tools/build_dist.sh` — для сборки `sign-file` используется локальный GCC вместо указанного Clang-варианта.
- `common/ack/arch/arm64/configs/gki_defconfig` — выключена автоматическая SCM-версия, настроен сертификат GrapheneOS как доверенный сертификат модулей; обязательная проверка подписей оставлена включённой.
- `common/ack/drivers/Kconfig` и `common/ack/drivers/Makefile` — подключена интеграция KernelSU.
- В `common/ack` добавлены исходники `KernelSU-Next/`, `drivers/kernelsu` и публичный сертификат GrapheneOS:
  `common/ack/certs/grapheneos-2026100201-module-signing-cert.pem`.

Финальный `.config` содержит, в частности:

```text
CONFIG_KSU=y
CONFIG_KPROBES=y
CONFIG_CFI_CLANG=y
CONFIG_MODULE_SIG_FORCE=y
CONFIG_SYSTEM_TRUSTED_KEYS="certs/grapheneos-2026100201-module-signing-cert.pem"
# CONFIG_LOCALVERSION_AUTO is not set
CONFIG_LOCALVERSION=""
CONFIG_SECURITY_SELINUX=y
CONFIG_LOCK_DOWN_KERNEL_FORCE_CONFIDENTIALITY=y
```

## 3. Совместимость модулей и сертификаты

`vermagic` у всех 327 локально собранных `.ko` одинаков и совпадает с официальными модулями, например:

```text
6.1.177-android14-11-gf6f62ba75ac9 SMP preempt mod_unload modversions aarch64RANDSTRUCT_7a73f06f7ac94e4e8edae1b45b26fc3ea19995cdbb613203c172887e37677017
```

В ядро встроен официальный публичный сертификат подписи модулей GrapheneOS:

```text
SHA-256 fingerprint: 7C:B1:24:FD:02:55:6B:C6:2E:49:3D:B7:00:E7:A5:AD:40:03:69:6E:0A:BE:FC:D7:4C:2B:14:94:07:FF:72:C2
serial: 0F250A6922FC0C9C382B579AB607A496FFE11625
```

Также в ядре присутствует сертификат, соответствующий подписи локальных KSU-модулей. Подпись менеджера KSU Next проверена отдельно: сертификат установленного APK имеет ожидаемые размер `998` байт и SHA-256 `79e590113c4c4c0c222978e413a5faa801666957b1212a328e46c00c69821bf7`. Ранняя проверка APK была неверно разобрана скриптом; после исправления парсера совпадение подтвердилось. Несовпадение подписи менеджера **не является причиной** текущей ошибки.

## 4. Официальный GrapheneOS-архив и образы

Официальный архив находится в:

```text
android-kernel-build/shiba-install-2026100201.zip
```

Его OpenSSH-подпись проверялась через `grapheneos-allowed_signers`; SHA-256 архива:

```text
b80aa1e3547507e60ca33bde7814608c313f78293afe2880520b6fc42feacc68
```

Из архива извлечены reference-образы. В частности:

```text
android-kernel-build/official-2026100201/init_boot.img
```

Это официальный `init_boot.img` для текущей версии: 8 МиБ, Android 17, fingerprint `2026100201`, SPL `2026-09-05`. Его SHA-256:

```text
e20a3af4ddb452cceb27f532e008b5bf7434a08f9081d15884f08c3106154c39
```

Распаковка показала: в официальном `init_boot.img` `kernel_size=0`, ramdisk около 2,76 МиБ. Основное ядро находится в `boot.img`; `init_boot.img` содержит generic ramdisk.

Официальный rollback reference для `boot_b`:

```text
android-kernel-build/ksu-next-3.4.0-2026100201-boot-64m-candidate/rollback-boot_b-official-2026100201.img
SHA-256: 5356af607e2d519ecd834a61303bbed456df60c8b0ef57d4c14f7235242073d4
```

Полный `flash-all.sh` из архива не запускался: он не является data-preserving способом отката.

## 5. Подготовка и проверка KSU `boot.img`

Первый локальный `out/shusky/dist/boot.img` имел размер 53 477 376 байт (51 МиБ), в отличие от физического раздела `boot_b` размером 67 108 864 байта (64 МиБ). В его AVB-footer была только development SPL property `2027-03-05`; это значение задаёт GKI build script для экспериментальных образов на разблокированных устройствах.

Сначала этот образ использовался **только** для временной загрузки:

```bash
fastboot boot android-kernel-build/kernel_pixel_6.1/out/shusky/dist/boot.img
```

Затем kernel payload извлекли, собрали boot header v4 и создали кандидат правильного размера с AVB-footer на 64 МиБ. Итоговый файл:

```text
android-kernel-build/ksu-next-3.4.0-2026100201-boot-64m-candidate/boot.img
size: 67 108 864 bytes
SHA-256: 35b5ba30fa998b96d0027e2f67cc73e7913f690a34e484b843ee2ebd3eb1085e
```

Ключевые команды упаковки:

```bash
python3 tools/mkbootimg/mkbootimg.py \
  --header_version 4 \
  --kernel /tmp/current-boot-unpack/kernel \
  --ramdisk /tmp/current-boot-unpack/ramdisk \
  --cmdline '' \
  --output /tmp/boot.raw.img

prebuilts/kernel-build-tools/linux-x86/bin/avbtool add_hash_footer \
  --image /tmp/boot.raw.img \
  --partition_name boot \
  --partition_size 0x4000000 \
  --prop com.android.build.boot.security_patch:2027-03-05
```

`avbtool verify_image` подтвердил целостность footer/hash для локального образа. Извлечённый kernel payload кандидата совпал с payload временно загруженного образа.

Кандидат сначала загрузили без записи в раздел:

```bash
adb reboot bootloader
fastboot getvar current-slot
fastboot getvar unlocked
fastboot boot android-kernel-build/ksu-next-3.4.0-2026100201-boot-64m-candidate/boot.img
```

Android загрузился; остался слот B, загрузились 316 модулей, включая `bcmdhd4398`, `gxp`, `mali_kbase`, `gs_thermal`. Затем обычная перезагрузка вернула устройство на штатный boot-раздел.

## 6. Изменения на устройстве

### Выполнено нами

Перед записью проверены hash кандидата, активный слот `b`, состояние `unlocked: yes` и размер `boot_b: 0x4000000`. Команда записи была только одна:

```bash
fastboot flash boot_b \
  android-kernel-build/ksu-next-3.4.0-2026100201-boot-64m-candidate/boot.img
fastboot reboot
```

`fastboot flash boot_b` завершилась `OKAY`. После перезагрузки Android загрузился на слоте B. `vendor_kernel_boot`, `init_boot`, `dtbo`, `vbmeta`, `vendor_dlkm` и `system_dlkm` этой командой не изменялись.

KSU-boot также скопирован в общую память телефона:

```bash
adb push <candidate>/boot.img \
  /sdcard/Download/boot-ksu-next-3.4.0-shiba-2026100201.img
```

SHA-256 на устройстве совпал с локальным кандидатом.

Официальный `init_boot.img` скопирован в:

```text
/sdcard/Download/2026100201_init_boot.img
```

### Выполнено пользователем

Пользователь самостоятельно прошил последний образ KSU Next, созданный менеджером:

```bash
adb pull /sdcard/Download/kernelsu_next_patched_20261005_131452.img
adb reboot bootloader
fastboot flash init_boot kernelsu_next_patched_20261005_131452.img
fastboot reboot
```

По выводу fastboot запись прошла в `init_boot_b`. Этот образ соответствует fingerprint `2026100201`; SHA-256 локальной копии:

```text
e09e7eb244a8189b648b045aeea84e6334169bd23deba394ea6d960aaadf624c
```

Более ранний файл `kernelsu_next_patched_20261005_131353.img` относится к fingerprint `2026100200`; он не использовался для записи.

## 7. Проверка KernelSU и причина отказа

После прошивки ядра система загружается, но менеджер сообщает, что KernelSU не установлен. Был запрошен и извлечён bugreport:

```text
android-kernel-build/KernelSU_Next_bugreport_2026-10-05_17_19.tar.gz
```

Извлечённые файлы:

```text
android-kernel-build/ksu-bugreport-2026-10-05-17-19/
```

`kernel_status.txt`:

```text
no_driver: kernel has no KernelSU-Next hook
manager_appid: 0
```

`basic.txt` фиксирует `KernelSU: -1`, менеджер v3.4.0 (33294), `LKM: false`. В `logcat.txt` многократно повторяется:

```text
KernelSU driver install syscall was blocked by seccomp
could not retrieve kernelsu driver fd
```

Причина подтверждается исходниками KernelSU Next:

- `userspace/ksud/src/ksucalls.rs` вызывает `libc::SYS_reboot` с KSU magic values, чтобы получить driver FD.
- `kernel/supercall/supercall.c` перехватывает этот вызов в `reboot_handler_pre()` и устанавливает FD.
- На GrapheneOS seccomp блокирует `reboot(2)` в процессе менеджера (`untrusted_app_34`) до того, как syscall доходит до kprobe ядра.

Поэтому наличие `CONFIG_KSU=y`, успешная загрузка ядра и загрузка штатных модулей сами по себе не доказывают, что KSU runtime работает. В данном случае driver FD не получен, `manager_appid=0`, root через KSU не функционирует. Дополнительный `init_boot` patch эту проблему не устранил.

**Исправление требует либо** изменить и пересобрать обе стороны протокола — KernelSU kernel и `ksud`/Manager APK — на syscall, который разрешён seccomp GrapheneOS, **либо** менять seccomp-политику самой ОС. Простая настройка приложения или ещё одна прошивка `boot`/`init_boot` проблему не устраняет. Глобально разрешать `reboot(2)` всем приложениям небезопасно.

## 8. Резервные копии и ограничения отката

Попытка считать разделы непосредственно с устройства не удалась: загрузчик Pixel запретил `fastboot fetch` для restricted partitions, например:

```text
Fetching restricted partition: boot_b is not allowed
```

Созданные нулевые placeholders были удалены. Пользователь сообщил, что собственные копии образов у него есть. Дополнительно сохранены официальные образы из подписанного архива, совпадающего с установленной версией. Это rollback references, но не raw-снимки разделов, считанные с телефона.

Для ручного возврата к официальным образам, если пользователь решит это сделать, локальные файлы:

```text
boot_b:     .../ksu-next-3.4.0-2026100201-boot-64m-candidate/rollback-boot_b-official-2026100201.img
init_boot_b:.../official-2026100201/init_boot.img
```

В этом отчёте команды восстановления не запускались. Полный install script и `fastboot -w` запрещены условиями сохранения данных.

## 9. Состояние на момент отчёта

Последняя read-only проверка через ADB:

```text
build=2026100201
slot=_b
kernel=6.1.177-android14-11-gf6f62ba75ac9
boot_completed=1
verifiedboot=orange
device_state=unlocked
```

Загружено 316 модулей. Менеджер `com.rifsxd.ksunext`, версия v3.4.0 / 33294, установлен, но его bugreport показывает отсутствие доступного KSU driver hook. Следовательно, сейчас устройство работает с экспериментальным `boot_b` и пользовательским `init_boot_b`, однако KSU root не работает.

## 10. Основные файлы и логи

```text
kernel repo:     android-kernel-build/kernel_pixel_6.1/
stock artifacts: android-kernel-build/stock-kernel-2026100201/
KSU build:       android-kernel-build/kernel_pixel_6.1/out/shusky/dist/
KSU boot image:  android-kernel-build/ksu-next-3.4.0-2026100201-boot-64m-candidate/boot.img
stock rollback:  android-kernel-build/ksu-next-3.4.0-2026100201-boot-64m-candidate/rollback-boot_b-official-2026100201.img
official archive:android-kernel-build/shiba-install-2026100201.zip
KSU bugreport:   android-kernel-build/KernelSU_Next_bugreport_2026-10-05_17_19.tar.gz
final build log: android-kernel-build/kernel_pixel_6.1/build-ksu-vermagic-final.log
```

В отчёте приведены основные команды и фактические результаты, восстановленные по сохранённым build-логам, артефактам, bugreport и истории сессии. Мелкие одноразовые диагностические команды могли не сохраниться дословно.

## 11. Addendum: экспериментальный переход supercall на `prctl` (2026-10-05)

После выяснения причины seccomp была проверена политика GrapheneOS Android 17 в `GrapheneOS/platform_bionic` branch `17`: `reboot` входит в `libc/SECCOMP_BLOCKLIST_APP.TXT`, тогда как `prctl` указан в `libc/SYSCALLS.TXT` и не указан в app blocklist. Поэтому `prctl(2)` выбран как новый транспорт KSU supercall; это пока гипотеза, которую ещё нужно подтвердить на самом процессе приложения.

Локально изменены исходники KernelSU Next:

- добавлен kprobe на `__arm64_sys_prctl` (и `__x64_sys_prctl` для x86_64), использующий обработчик supercall; старый reboot kprobe оставлен для совместимости;
- обращения за KSU driver FD в `userspace/ksud/src/ksucalls.rs` и `userspace/ksuinit/src/lib.rs` переведены с `SYS_reboot` на `SYS_prctl` с пятым аргументом `0`;
- вызовы в `userspace/ksud/src/susfsd.rs` также переведены на `SYS_prctl`.

Собран side-by-side debug Manager с package ID `com.rifsxd.ksunext.prctl`; он не заменяет установленный официальный Manager. Для debug Manager используется локальный debug signer, поэтому его сертификат намеренно закреплён в kernel build:

```text
KSU_MANAGER_PACKAGE=com.rifsxd.ksunext.prctl
KSU_NEXT_MANAGER_SIZE=0x2e8
KSU_NEXT_MANAGER_HASH=729b3cc73dd52b1eb06e5f99ade350c097827e3976c2f8bf26ca68c354d69239
```

Debug APK SHA-256:

```text
02e5be5370c26f4da44dd60a32ca0199fbc8b81b4ebdf486260701370fe75ae3
```

Kernel-сборка с новым transport завершилась успешно (`build-ksu-prctl-final.log`, 121 action); все 327 локальных модулей сохранили штатный `vermagic`. Новый 64-МиБ кандидат:

```text
android-kernel-build/ksu-next-3.4.0-2026100201-prctl-candidate/boot.img
SHA-256: a4eb02da4efefafb8c98ee695901190965a051f5fb26874f3c159d7e21a92538
```

AVB footer и hash кандидата проверены локально. После того как телефон снова определился по ADB, сначала выполнена временная загрузка через `fastboot boot`, без записи раздела. Side-by-side APK установлен под package ID `com.rifsxd.ksunext.prctl`; официальный `com.rifsxd.ksunext` оставлен установленным.

Временно загруженное ядро прошло boot, затем запущенный от UID debug Manager `ksud` выполнил `debug version` и вывел `Kernel Version: 33294`. Это подтверждает, что syscall `prctl` прошёл seccomp, kernel hook выдал driver FD, а ioctl к KSU-драйверу сработал. В logcat тестового запуска не обнаружены `KernelSU driver install syscall was blocked`, `could not retrieve kernelsu driver fd` или `SIGSYS`.

После успешного временного теста образ записан **только в `boot_b`** командой `fastboot flash boot_b ...`; fastboot подтвердил размер раздела `0x4000000` и успешную запись. Телефон перезагрузился, `sys.boot_completed=1`, сборка осталась `2026100201`, активный слот `_b`; повторный запуск `ksud debug version` снова вернул `33294`. `init_boot_b` и остальные разделы не трогались; `fastboot -w` не использовался, данные не стирались.

## 12. Анализ логов и дополнительная правка Manager (2026-10-05)

Получены файлы из `Download`:

```text
KernelSU_Next_install_log_2026-10-05-21-13-59.log
KernelSU_Next_bugreport_2026-10-05_21_13.tar.gz
```

Установочный лог показывает попытку использовать LKM-установщик и ошибку `asset not found: android14-6.1_kernelsu.ko`. Это отдельный неуспешный сценарий установки модуля: данное ядро собрано с `CONFIG_KSU=y`, KernelSU встроен, поэтому этот LKM-файл не ожидается и повторять этот экран установки не нужно.

Bugreport был создан до исправления JNI Manager и потому закономерно фиксирует `KernelSU: -1` и `no_driver`. Исходный тест `ksud debug version` доказывал только, что Rust `ksud` через `prctl` получает FD и читает версию; этого было недостаточно для Manager UI. В `manager/app/src/main/cpp/ksu.cc` добавлен такой же `KSU_INSTALL_MAGIC` запрос через `SYS_prctl`, после чего собран и установлен обновлённый APK с прежними package ID и сертификатом, совпадающим с закреплённым в ядре (`729b3cc7…54d69239`).

Дальнейшая проверка выявила причину ошибок `EPERM`: `apk_sign.c` в kernel принимает v2 signing block, но отвергает неизвестный v3 блок (`Unexpected v3 signature found!`). Поэтому исходный debug APK с ожидаемым сертификатом не проходил kernel-проверку. Также первая ручная JNI-сборка случайно зависела от `libc++_shared.so`, которой нет в APK; JNI пересобрана со статическим C++ runtime.

Собран final side-by-side APK с JNI `prctl` и **v2-only** подписью тем же сертификатом, который закреплён в ядре. Он установлен через `adb install -r`. Проверки на телефоне:

```text
ksud debug get-sign <installed base.apk>
  size: 0x2e8, hash: 729b3cc7...54d69239
ksud debug version
  Kernel Version: 33294
C++ JNI probe
  version=33294 uapi=4 manager_appid=0
```

То есть обе стороны получают driver FD через `prctl`, а APK-формат/сертификат соответствует kernel parser. После обычной перезагрузки `manager_appid` всё ещё 0, а `feature check su_compat` возвращает `EPERM`; kernel не регистрирует тестовый Manager. Новых flash-команд после установки `boot_b` не выполнялись.

### 13. Проверка новой порции bugreport (2026-10-05)

Bugreport `KernelSU_Next_bugreport_2026-10-05_22_47.tar.gz` создан **официальным** пакетом `com.rifsxd.ksunext` (UID 10249), не тестовым `com.rifsxd.ksunext.prctl` (UID 10265). Путь в logcat `/data/app/.../com.rifsxd.ksunext-...` это подтверждает. Поэтому `KernelSU: -1`/`no_driver` в этом отчёте не являются проверкой debug-варианта; официальный Manager ожидаемо не совпадает с пакетом, закреплённым в экспериментальном ядре.

Чтобы различать одинаковые иконки, debug-вариант переименован в `KernelSU-Next (PRCTL test)`, установлен и запущен явно. В его logcat видны `ksud debug su` → `EPERM`, затем отдельная ошибка libsu/ART:

```text
SecurityException: Writable dex file
/data/user_de/0/com.rifsxd.ksunext.prctl/cache/main.jar is not allowed
```

Это отдельная проблема root-service на Android 17. Непосредственный KSU JNI probe тестового пакета даёт `version=33294`, `uapi=4`, `flags=0`, `manager_appid=0`; значит syscall и драйвер доступны, но kernel не считает приложение Manager. Правильный bugreport `KernelSU_Next_bugreport_2026-10-05_23_03.tar.gz` создан именно `com.rifsxd.ksunext.prctl` (UID 10265), и в `basic.txt` теперь `KernelSU: 33294`. Его `kernel_status.txt` всё ещё содержит `no_driver`, но эта строка захардкожена в `getBugreportFileUnrooted()` и не проверяет результат; ей противоречит успешное получение версии. `ksud debug get-sign` подтвердил размер сертификата `0x2e8` и ожидаемый SHA-256; seccomp-block/SIGSYS нет. В logcat `KSU_IOCTL_GET_FEATURE` и `debug su` продолжают завершаться с `EPERM`. Следующая работа — выяснить, почему kernel-side package tracker не назначает manager appid; затем отдельно устранить libsu writable-dex отказ.

### 14. Исправление начального сканирования Manager (2026-10-06)

Встроенный режим `CONFIG_KSU=y` при `on_post_fs_data()` запускал `ksu_observer_init()`, но не выполнял первоначальный `track_throne(false)`. Поздно загружаемый LKM-путь такой scan делал, а built-in путь — нет; он зависел от будущего события `/data/system/packages.list`. Это объясняет, почему валидный APK оставался без manager appid после загрузки.

Добавлен начальный `track_throne(false)` сразу после запуска observer в `kernel/runtime/boot_event.c`. Kernel build успешно завершён. Подготовлен отдельный кандидат (без записи на телефон):

```text
android-kernel-build/ksu-next-3.4.0-2026100201-manager-scan-fix-candidate/boot.img
SHA-256: a711131518de1ba17cef735d92953c248b1fd3b7157cda1cb3ec4f55859dd2fd
Размер: 64 MiB; AVB footer/hash проверен avbtool; kernel release совпадает с установленным.
```

Кандидат временно загружен через `fastboot boot` с разрешения пользователя; ни один раздел не записывался. Драйвер по-прежнему выдаёт `Kernel Version 33294`, но `flags=0`, `manager_appid=0`, feature ioctl и `ksud debug su` возвращают `EPERM`. Значит одного сканирования в post-fs-data недостаточно. После обычной перезагрузки телефон вернулся на постоянный `boot_b`; разделы и данные не менялись.

### 15. Повторная попытка скана после boot-completed

Свежие bugreport-файлы от `23:48` и `00:01` принадлежат правильному тестовому пакету. Они подтверждают состояние выше; `no_driver` в `kernel_status.txt` по-прежнему невалидно, поскольку жёстко прописано в unrooted bugreport helper. Чтобы повторить scan после того, как PackageManager точно сформировал `packages.list`, `on_boot_completed()` изменён с `track_throne(true)` на `track_throne(false)`; первоначальный scan в `on_post_fs_data()` сохранён.

Вторая сборка успешна; кандидат с повторным scan после boot-completed был временно загружен с разрешения пользователя (без записи раздела). На нём direct probe снова показал `version=33294`, `flags=0`, `manager_appid=0`, feature ioctl — `EPERM`. Bugreport `KernelSU_Next_bugreport_2026-10-06_00_56.tar.gz` подтверждает `KernelSU: 33294` и `ksud debug su` → `EPERM`; значит повторный scan на boot-completed также не решил регистрацию. После теста выполнена обычная перезагрузка, устройство вернулось на постоянный `boot_b`.

Кандидат второго этапа:

```text
android-kernel-build/ksu-next-3.4.0-2026100201-manager-scan-bootcompleted-candidate/boot.img
SHA-256: f22fec6ab3d22a8359b486e650d20b957e96ccefaff9fa30da2fe1448e1e7f30
Размер: 64 MiB
```

Дальше не следует делать очередной blind scan/reflash. Нужна диагностика самого tracker: подтвердить вызовы observer/scan, чтение `packages.list`, обход `/data/app`, извлечение имени пакета и результат kernel APK v2 verifier. Текущие KSU bugreport-логи этих kernel `pr_info` не содержат.

### 16. Диагностическая сборка tracker

Чтобы не собирать полный системный bugreport с лишними пользовательскими данными, подготовлена временная read-only диагностика в верхних зарезервированных битах `GET_INFO.flags`, только когда задан тестовый `KSU_MANAGER_PACKAGE`. Она показывает стадии: `track_throne` вызван; `packages.list` открыт; target package присутствует в списке; scan запущен; `/data/app` открыт; встречены подкаталоги и `base.apk`; имя APK-пути совпало; kernel v2 signer прошёл; UID пакета найден и manager appid назначен; scan завершился. Менеджерские полномочия это не меняет.

Сборка успешна, образ подготовлен, AVB проверен и с разрешения пользователя временно загружен. GET_INFO вернул `flags=0x0`, а feature ioctl остался `EPERM`. Дизассемблирование `do_get_info` в именно этой сборке подтверждает, что верхние флаги содержат `tracker_diag_state << 8`; диагностические функции tracker также присутствуют в vmlinux. Следовательно, во время этой загрузки ни один путь не вызвал `track_throne()` — сбой происходит раньше обхода каталогов и проверки APK/подписи. Это исключает гипотезу, что текущая проблема обязательно в сертификате.

```text
android-kernel-build/ksu-next-3.4.0-2026100201-tracker-diag-candidate/boot.img
SHA-256: 3e8f61131a94a61d6b6bd21f08c8e8fe5889fa87d7a90173c9d4ff697e708baf
Размер: 64 MiB
Результат probe: version=33294 flags=0x0 manager_appid=0
```

После теста выполнена обычная перезагрузка; устройство вернулось на постоянный `boot_b`. Для следующего шага нужно различить, почему не отрабатывает `post-fs-data`/`boot-completed`/zygote hook, и затем обеспечить надёжный scan после готовности `/data`. Дополнительная временная загрузка пока не выполнялась.

### 17. Исправление MTE-тега указателя в execve hook (2026-10-06)

Полный системный bugreport был получен локально, чтобы сохранить kernel log: `system-bugreport-20261006/bugreport.zip`. В предыдущей загрузке он содержал 418 `Access filename failed for execve_handler_pre`, без `exec zygote`/`on_post_fs_data`; `ro.sanitize.memtag_heap=true`. Это подтверждает отказ чтения user string в execve hook, но связь именно с MTE остаётся объяснением механизма, не отдельным прямым измерением.

В исходном дереве уже оказались удалены `untagged_addr()` в перечисленных user-pointer чтениях (`ksud_integration.c`, `syscall_event_bridge.c`, `sucompat.c`, `adb_root.c`, `sulog/event.c`). Отдельный временный fallback scan, который я добавлял локально во время диагностики, признан неверным обходом и удалён; он **не был успешно собран, не загружался и не прошивался**. Ошибочный незавершённый build останавливался на `ksu_cred` undeclared.

Собран тестовый образ с исправлением untagged pointers. В AVB сохранён официальный уровень security patch `2026-09-05`, а не значение build-dist `2027-03-05`. Образ был только временно загружен через `fastboot boot`; записи разделов не было. В свежем системном dmesg:

```text
[9.599s]  exec zygote, /data prepared, second_stage: 1
[9.599s]  on_post_fs_data!
[19.575s] Searching manager...
[19.582s] sha256: 729b...d69239, expected: 729b...d69239
[19.582s] Found .../com.rifsxd.ksunext.prctl-.../base.apk, is_manager: 1
[19.582s] Crowning manager: com.rifsxd.ksunext.prctl(uid=10265)
```

В этом dmesg — **0** `Access filename failed for execve_handler_pre`. JNI probe подтвердил `version=33294 flags=0x2 manager_appid=10265`; `feature check su_compat` вернул `supported`. После завершившей тест обычной перезагрузки, на текущей загрузке probe по-прежнему показывает `flags=0x2 manager_appid=10265`, и feature ioctl больше не возвращает `EPERM`. Пользователь подтвердил наличие root.

Тестовый образ:

```text
android-kernel-build/ksu-next-3.4.0-2026100201-mte-untagged-candidate/boot.img
SHA-256: ed42ca079105a68c3d15fcab6f3074635b762bd6865d48d5853ea3c458f804ec
Размер: 64 MiB; AVB footer/hash проверены; security_patch property 2026-09-05.
```

Эти результаты заменяют предыдущие гипотезы в разделах 7, 12, 14–16: seccomp `reboot` был наблюдаемым симптомом, не первопричиной; в3-подпись не доказана как причина; tracker не запускался, потому что execve hook не мог прочитать имя. Пока подтверждён тестовый пакет `com.rifsxd.ksunext.prctl`; официальный Manager и возврат к штатному `reboot` transport ещё не проверялись. `init_boot_b` не менялся, поэтому фактор `ksuinit` остаётся непроверенным.

### 18. Проверка ADB/Termius и NeoBackup (дополнение)

Проверка root из ADB выполнялась командой:

```sh
adb -s 38221FDJH006CS shell su -c 'id; id -u'
```

В ранней попытке `/system/bin/su` отсутствовал, а обычная ADB shell имела UID 2000. Это не означало отсутствия root: KernelSU делает sucompat-перехват для разрешённого UID, а физический файл `/system/bin/su` не обязателен. Позже проверка через `adb shell su -c ...` успешно дала `uid=0`, контекст `u:r:ksu:s0`; Termius (`uid=10201`) также проходил через `sucompat`. Отдельный запрос `adb root` в одном из ранних состояний отвечал `adbd cannot run as root in production builds`; это относится к root-режиму самого `adbd`, не к `su` в ADB shell. Не считать эти две проверки эквивалентными.

Для NeoBackup (`com.machiav3lli.backup`, UID 10193) kernel log показал сохранённое разрешение `allow: 1`. Bugreport `neo-backup-selinux-test/bugreport.zip` содержит попытки `sys_execve su found` и выдачу KSU fd; логи самого NeoBackup сообщали `libsu uses root shell = true`. Его проверка, однако, завершалась `127`:

```text
sh '/data/user/0/com.machiav3lli.backup/files/assets/plugin/checkroot.internal_sh'
Permission denied
is NOT like root
```

В том же bugreport SELinux вернули в Enforcing в `14:07:58`; ошибка скрипта была около `14:08:16`, а переключение в permissive произошло только в `14:08:29`. То есть именно эта неудачная проверка зафиксирована в Enforcing, несмотря на последующий permissive-тест.

**Финальный пользовательский результат:** NeoBackup заработал после добавления ему необходимых capabilities в профиле KernelSU. Точные названия добавленных capabilities в истории не указаны. Termius и ADB также работают по сообщению пользователя.

В ответ на просьбу «исправляем и прошиваем» локально добавлялся fallback `untagged_addr()` только в ADB Root path, и полная сборка прошла. После уточнения, что Termius и ADB уже работают, этот локальный исходный патч был отменён. Сборка/её артефакты остались только на компьютере: **этот fallback-образ не загружался и не прошивался**. Никаких новых записей разделов после временного MTE-теста не было; активный слот оставался `_b`.

Основные команды из сессии:

```sh
# Shusky build
KLEAF_REPO_MANIFEST=aosp_manifest.xml ./build_shusky.sh --lto=full

# ADB su test
adb -s 38221FDJH006CS shell su -c 'id; id -u'

# ADB daemon-root test (отдельно от su)
adb -s 38221FDJH006CS root
adb -s 38221FDJH006CS wait-for-device
adb -s 38221FDJH006CS shell id

# Full system bugreport, then local extraction/filtering of relevant KSU lines
adb -s 38221FDJH006CS bugreport <local-output-directory>/bugreport.zip
```

Другие выполненные действия и кандидаты описаны выше; markdown — хронология основных диагностических этапов, не дословная транскрипция каждого вызова оболочки. Детальные логи и результаты сборок остаются по указанным в отчёте локальным путям.

### 19. Запись проверенного MTE-кандидата в `boot_b` (2026-10-06)

По явному разрешению пользователя образ, который ранее успешно проверяли через `fastboot boot`, записан только в `boot_b`. Перед записью проверены SHA-256, AVB footer/hash и security patch property `2026-09-05`; текущий слот был `_b`, bootloader сообщил `unlocked: yes`.

```sh
fastboot -s 38221FDJH006CS flash boot_b \
  android-kernel-build/ksu-next-3.4.0-2026100201-mte-untagged-candidate/boot.img
# Sending boot_b OKAY; Writing boot_b OKAY
fastboot -s 38221FDJH006CS reboot
```

Патчен только `boot_b`; `init_boot_b`, другие разделы и пользовательские данные не трогались; `-w` не запускался. Сразу после `fastboot reboot` устройство временно не перечислялось по USB, но затем пользователь подтвердил загрузку. Повторная проверка через ADB подтвердила `sys.boot_completed=1`, слот `_b`, build `CP3A.260905.009`, kernel `6.1.177-android14-11-gf6f62ba75ac9`; `adb shell su -c id` вернул UID 0, контекст `u:r:ksu:s0`. Дальнейших команд записи/переключения слота не выполнялось.

### 20. Патчи рядом с отчётом-инструкцией

Подготовлены пять исходных diff-патчей и README в `android-kernel-build/patches/`, рядом с этим отчётом. Теги и базы: kernel workspace `2026100200` / `105c7cdc8e1139d83a1d8cc2dc56513e432db834`, ACK `2026100200` / `f6f62ba75ac9dbcf802b2ca2ebce2058362a04d6`, KernelSU-Next `v3.4.0` / `1a879d6a866f80b1fa1c1009a2ffa747873cbb5e`; tested OS build — GrapheneOS `2026100201`. Патчи проверены на обратную применимость к текущим worktree. Состав, точные каталоги применения и host-specific overrides описаны в `patches/README.md`.

### 21. Собранные бинарники рядом с отчётом-инструкцией

В `android-kernel-build/artifacts/` скопированы проверенный и фактически прошитый `boot_b` образ, тестовый PRCTL Manager APK, установленный regular Manager APK, matching stock boot для отката и публичный модульный сертификат GrapheneOS. SHA-256, назначения и происхождение каждого файла указаны в `artifacts/README.md`. Исходные build artifacts оставлены на прежних местах.

### 22. Полная инструкция установки/сборки

Создан `INSTALL.md`. В нём описаны clone точных ревизий, применение патчей в соответствующих репозиториях, сборка/AVB-перепаковка, установка на активный boot slot, проверка root, Neo Backup профиль и ограничения OTA/rollback. Инструкция привязана к GrapheneOS `2026100201`; она предупреждает не прошивать этот образ на иные builds. Отдельная пошаговая OTA/KernelSU update procedure вынесена в `UPDATE.md`; в ней учтено, что на тестовом телефоне ранее изменялся `init_boot_b`.

