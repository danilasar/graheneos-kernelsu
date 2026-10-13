# Установка KSU 3.4.1 на Pixel 8 / GrapheneOS 2026100601

Только `shiba`, fingerprint `google/shiba/shiba:17/CP3A.261005.005/2026100601:user/release-keys`, SPL `2026-10-05`, kernel `6.1.177-android14-11-g5969734141aa`. KSU встроен (`CONFIG_KSU=y`), не отдельный устанавливаемый `.ko`.

## 1. Готовый комплект: подготовка

Выполняй команды из корня этого репозитория. Укажи USB serial, а не выбирай первое устройство: могут одновременно отображаться USB и беспроводной ADB.

```sh
export SERIAL=YOUR_PIXEL_SERIAL
export RELEASE_DIR="$PWD"
(cd artifacts/2026100601 && sha256sum -c SHA256SUMS)
adb -s "$SERIAL" shell 'getprop ro.product.device; getprop ro.build.fingerprint; getprop ro.build.version.security_patch; getprop ro.boot.slot_suffix; getprop ro.boot.vbmeta.device_state'
```

При несовпадении ОС/модели остановись. Загрузчик должен уже быть разблокирован; повторный unlock не нужен. Unlock стирает данные и не входит в эту процедуру.

Установи исправленный менеджер **до загрузки нового ядра**:

```sh
adb -s "$SERIAL" install -r artifacts/2026100601/Manager-prctl341-with-ksud.apk
```

Пакет — `com.rifsxd.ksunext.prctl341`, сертификат SHA256 `b4eb1c14210fff0e1d8102e0f54a9cd1dfd674050dc3a520ff88dc8c048b0da3`. Не заменять обычным Manager. APK должен содержать `lib/arm64-v8a/libksud.so`: одного JNI драйвера недостаточно.

## 2. Временная загрузка и проверка

```sh
adb -s "$SERIAL" reboot bootloader
fastboot -s "$SERIAL" getvar unlocked
fastboot -s "$SERIAL" getvar current-slot
fastboot -s "$SERIAL" boot artifacts/2026100601/boot-ksu341.img
```

После возвращения ADB проверь:

```sh
adb -s "$SERIAL" shell 'getprop sys.boot_completed; getprop ro.boot.bootreason; getprop sys.boot.reason; cat /proc/uptime; getprop ro.build.version.incremental; getprop ro.boot.slot_suffix; uname -r; getenforce; wc -l /proc/modules'
adb -s "$SERIAL" shell monkey -p com.rifsxd.ksunext.prctl341 -c android.intent.category.LAUNCHER 1
```

Ожидаются build `2026100601`, завершённая загрузка, Enforcing, работающий Manager v3.4.1/33333/UAPI5 и root. На двух тестовых телефонах загрузились 316 модулей, включая Wi-Fi/GPU/GXP/thermal. Количество само по себе не заменяет проверку нужных модулей и функций.

**Не считать boot_completed и uname доказательством:** после неудачного временного boot телефон может перезагрузиться на штатное ядро. Watchdog boot reason, сброс uptime, недоступный драйвер — повод остановиться. В bugreport различай `KERNEL LOG (dmesg)` текущей загрузки и `LAST KMSG` предыдущей. Обычная перезагрузка после `fastboot boot` возвращает образ из раздела, а не кандидат.

Независимая проверка root через отладочный Manager (не выдаёт разрешение UID2000):

```sh
BASE=$(adb -s "$SERIAL" shell pm path com.rifsxd.ksunext.prctl341 | head -1 | tr -d '\r' | cut -d: -f2)
LIB="${BASE%/base.apk}/lib/arm64/libksud.so"
adb -s "$SERIAL" shell run-as com.rifsxd.ksunext.prctl341 "$LIB" debug version
printf 'id\ngetenforce\nexit\n' | adb -s "$SERIAL" shell "run-as com.rifsxd.ksunext.prctl341 '$LIB' debug su"
```

Ожидаются `Kernel Version: 33333`, `uid=0`, `u:r:ksu:s0`, Enforcing. `adb shell su -c id` может не работать на чистом телефоне до отдельного разрешения shell UID2000. `adb root` — другая функция production adbd.

Проверь нужные приложения. Neo Backup требует отдельного root-профиля; прежний рабочий профиль использовал все capabilities, но минимальный набор не установлен. Не расширять права автоматически и не переводить SELinux в permissive.

## 3. Запись после успешного теста

Только после проверки и явного разрешения владельца снова войди в bootloader. Ещё раз проверь слот, unlocked и размер **конкретного** раздела.

```sh
adb -s "$SERIAL" reboot bootloader
fastboot -s "$SERIAL" getvar current-slot
fastboot -s "$SERIAL" getvar unlocked
# Только если current-slot: a:
fastboot -s "$SERIAL" getvar partition-size:boot_a
fastboot -s "$SERIAL" flash boot_a artifacts/2026100601/boot-ksu341.img
fastboot -s "$SERIAL" reboot
```

Для текущего B используй только `boot_b`, не обе команды. Размер раздела `0x4000000`. После постоянной загрузки повтори проверки версии, root, boot reason и модулей. Не трогать `init_boot`, `vendor_boot`, `vendor_kernel_boot`, `dtbo`, `vbmeta` или dlkm-разделы. Не менять слот.

## 4. Сборка из исходников

Нужны Linux x86_64, Git, Python3, GCC/OpenSSL headers, LZ4, Rust с Android target, Android SDK/NDK29, Java21. Свободное место контролируй; полные build caches занимают гигабайты.

```sh
git clone --branch 2026100600 https://gitlab.com/GrapheneOS/kernel_pixel_6.1.git kernel_pixel_6.1
cd kernel_pixel_6.1
git checkout 6e6affd1156ffe7f564f8f65f6d923be8924a7b9
git submodule update --init common/ack
# common/ack HEAD обязан быть 5969734141aa492032b4bcab559c704e064fe7cf
git clone https://github.com/KernelSU-Next/KernelSU-Next.git common/ack/KernelSU-Next
git -C common/ack/KernelSU-Next checkout 8f902aeb16033024143ae36cf711df902cf02fae
```

Не использовать shallow KSU без точной настройки версии; sandbox не содержит вложенные Git metadata. Применение пяти патчей и копирование исходников описаны в [patches/2026100601/README.md](patches/2026100601/README.md). Настройки GCC host-specific, их нужно проверить.

### 4.1. Полноценный APK

Выполняй из `common/ack/KernelSU-Next`; укажи свой SDK/NDK и путь к host libclang. Сборка ARM64 без необязательных LKM образов предназначена для нашего встроенного KSU, не для patch/install LKM экранов.

```sh
export ANDROID_HOME="$HOME/Android/Sdk"
export ANDROID_NDK_HOME="$ANDROID_HOME/ndk/29.0.14206865"
export PATH="$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin:$PATH"
export CARGO_TARGET_AARCH64_LINUX_ANDROID_LINKER=aarch64-linux-android35-clang
export CC_aarch64_linux_android=aarch64-linux-android35-clang
export AR_aarch64_linux_android=llvm-ar
export LIBCLANG_PATH=/path/to/host/libclang/directory
export BINDGEN_EXTRA_CLANG_ARGS_aarch64_linux_android="--sysroot=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/sysroot --target=aarch64-linux-android35"
rustup target add aarch64-linux-android
(cd userspace/ksud && cargo build --locked --release --target aarch64-linux-android)
mkdir -p manager/app/src/main/jniLibs/arm64-v8a
cp userspace/ksud/target/aarch64-linux-android/release/ksud manager/app/src/main/jniLibs/arm64-v8a/libksud.so
(cd manager && JAVA_HOME=/path/to/java-21 ./gradlew :app:assembleDebug)
```

Проверь ZIP содержимое и `apksigner verify --print-certs`. Включена preBuild-проверка наличия arm64 ksud. Но она проверяет наличие, не происхождение: используй именно только что собранный PRCTL бинарник.

**Подпись:** локальный новый debug key не совпадёт с опубликованным APK. Для воспроизведения с новым ключом сначала собери APK, извлеки DER сертификат, вычисли его размер и SHA256, затем замени ожидаемые package/hash/size в `build/kernel/_setup_env.sh` (патч 0005) и пересобери ядро. Закрытый ключ опубликованного Manager не включён. Старый опубликованный boot работает только с опубликованным совместимым APK.

### 4.2. Сертификат и сборка ядра

Публичный `artifacts/2026100601/module-signing-cert.pem` извлечён из официального boot этой ОС. DER SHA256 `9ca70bf606dcf3c61b91340d63cdfdb709168a0ad9a6f6ee723181bdef4af9cd`. Не использовать старый сертификат от `2026100201`; несовпадение приводит к отказу модулей и раннему watchdog.

Из корня kernel workspace:

```sh
KLEAF_REPO_MANIFEST=aosp_manifest.xml tools/bazel run --config=stamp --config=shusky //private/devices/google/shusky:shusky/dist --disable_slab_canary --lto=full
```

Эта команда создаёт dist; отдельный postprocessing wrapper для подписи локальных `.ko` не запускался при финальной сборке. Локальные `.ko` не устанавливаются на телефон: используем штатные подписанные модули. Проверь итоговый `.config`: KSU=y, MODULE_SIG_FORCE=y, правильный SYSTEM_TRUSTED_KEYS; DER сертификат должен фактически присутствовать в `out/shusky/dist/Image`. Проверяй полный vermagic, включая RANDSTRUCT, и ABI; подмена suffix не обеспечивает совместимость.

### 4.3. Упаковка

Не прошивай промежуточный dist/boot.img напрямую:

```sh
mkdir -p out/ksu-final/unpacked
python3 tools/mkbootimg/unpack_bootimg.py --boot_img out/shusky/dist/boot.img --out out/ksu-final/unpacked
python3 tools/mkbootimg/mkbootimg.py --header_version 4 --os_version 17.0.0 --os_patch_level 2026-10 --kernel out/ksu-final/unpacked/kernel --ramdisk out/ksu-final/unpacked/ramdisk --cmdline '' --output out/ksu-final/boot.img
prebuilts/kernel-build-tools/linux-x86/bin/avbtool add_hash_footer --image out/ksu-final/boot.img --partition_name boot --partition_size 0x4000000 --prop com.android.build.boot.security_patch:2026-10-05
prebuilts/kernel-build-tools/linux-x86/bin/avbtool verify_image --image out/ksu-final/boot.img
prebuilts/kernel-build-tools/linux-x86/bin/avbtool info_image --image out/ksu-final/boot.img
stat -c %s out/ksu-final/boot.img
sha256sum out/ksu-final/boot.img
```

Размер 67108864. AVB Algorithm NONE: проверка подтверждает целостность hash/footer, **не официальную подпись или право блокировать загрузчик**. SHA пересборки может отличаться (в том числе из-за соли AVB). Заголовок и SPL всегда сверяй со штатным образом целевой ОС.
