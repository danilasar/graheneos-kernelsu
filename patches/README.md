# Набор патчей KernelSU Next для GrapheneOS на Pixel 8

Патчи лежат рядом с инструкцией по установке и бинарными файлами. Теги и ревизии, использованные для проверенной сборки:

```text
Сборка GrapheneOS на устройстве: 2026100201 (уровень исправлений безопасности 2026-09-05)
Рабочее дерево ядра GrapheneOS:  тег 2026100200, коммит 105c7cdc8e1139d83a1d8cc2dc56513e432db834
Подмодуль ACK common/ack:        тег 2026100200, коммит f6f62ba75ac9dbcf802b2ca2ebce2058362a04d6
KernelSU-Next:                   тег v3.4.0, коммит 1a879d6a866f80b1fa1c1009a2ffa747873cbb5e
```

Проверенный выпуск на устройстве — `2026100201`, а доступные теги исходников — `2026100200`; это различие описано в отчёте.

Где и в каком порядке применять патчи:

```sh
# В common/ack/KernelSU-Next (после клонирования и перехода на базовый коммит KSU):
git apply /path/to/patches/0001-mte-user-pointer-reads.patch
git apply /path/to/patches/0002-grapheneos-prctl-transport.patch
git apply /path/to/patches/0003-prctl-test-manager-build.patch

# В common/ack:
git apply /path/to/patches/0004-ack-kernelsu-integration.patch
ln -s ../KernelSU-Next/kernel drivers/kernelsu
cp /path/to/artifacts/grapheneos-2026100201-module-signing-cert.pem \
  certs/grapheneos-2026100201-module-signing-cert.pem

# В корне рабочего дерева ядра:
git apply /path/to/patches/0005-local-build-overrides.patch
```

## Содержимое

1. `0001-mte-user-pointer-reads.patch` — убирает преобразование `untagged_addr()` из затронутых чтений пользовательских указателей в KernelSU.
2. `0002-grapheneos-prctl-transport.patch` — добавляет путь супервызова через `prctl` и обновляет ksud, ksuinit, susfsd и JNI-часть менеджера.
3. `0003-prctl-test-manager-build.patch` — настройки отладочного пакета и сборки менеджера, нужные только для теста. Локально извлечённые JNI-библиотеки в патч не входят; используй приложенный APK, если не пересобираешь именно этот тестовый APK.
4. `0004-ack-kernelsu-integration.patch` — подключает KernelSU в Kconfig и Makefile и задаёт в ACK доверенный сертификат модулей GrapheneOS.
5. `0005-local-build-overrides.patch` — локальные обходные решения для Kleaf и инструментов сборки, с которыми получена эта сборка. Патч содержит путь к заголовкам GCC, зависящий от сборочной машины (`/usr/lib64/gcc/x86_64-alt-linux/13/`); на другом дистрибутиве Linux его нужно поправить, а не применять вслепую.

Исходники KernelSU должны лежать в `common/ack/KernelSU-Next`, а `drivers/kernelsu` должен быть символической ссылкой на их каталог `kernel`. Сертификат, который копируется выше, публичный. Закрытый ключ подписи APK менеджера в комплект не входит; для установки используй готовый тестовый APK менеджера из `../artifacts/`.

Привилегии, которые нужны NeoBackup, задаются в профиле приложения в KernelSU уже на работающей системе, а не патчем к исходникам. Их точные названия в сессии не записаны; настрой их так же, как на проверенном телефоне. Перепаковка загрузочного образа Android в набор патчей не входит; см. инструкцию по установке. Итоговый загрузочный образ лежит в `../artifacts/`.
