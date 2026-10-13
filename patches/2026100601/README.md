# Патчи GrapheneOS 2026100601 / KSU3.4.1

Базы: workspace6e6affd1156ffe7f564f8f65f6d923be8924a7b9, ACK5969734141aa492032b4bcab559c704e064fe7cf, KSU8f902aeb16033024143ae36cf711df902cf02fae. Не применять root patches для3.4.0 к этой базе. Патчи проверены на обратную применимость к итоговым рабочим деревьям.

Из точного KSU repository применить0001–0003:

```sh
cd /path/to/kernel_pixel_6.1/common/ack/KernelSU-Next
git apply "$RELEASE_DIR/patches/2026100601/0001-mte-user-pointer-reads.patch"
git apply "$RELEASE_DIR/patches/2026100601/0002-grapheneos-prctl-transport.patch"
git apply "$RELEASE_DIR/patches/2026100601/0003-prctl-manager-build.patch"
```

0001: user-pointer/MTE reads. 0002: kernel PRCTL + Rust userspace/JNI, включая диагностические логи проверенной сборки. 0003: отдельный package/label и guard на наличие arm64 libksud.so. Бинарник ksud не входит в diff: его **обязательно собрать и скопировать** до Gradle (INSTALL.md).

В ACK:

```sh
cd /path/to/kernel_pixel_6.1/common/ack
git apply "$RELEASE_DIR/patches/2026100601/0004-ack-kernelsu-integration.patch"
cp "$RELEASE_DIR/artifacts/2026100601/module-signing-cert.pem" certs/grapheneos-2026100601-module-signing-cert.pem
# Для свежего дерева, где drivers/kernelsu ещё не существует:
cp -a KernelSU-Next/kernel drivers/kernelsu
cp -a KernelSU-Next/uapi drivers/kernelsu/uapi
```

Kernel include/uapi symlink ожидает ../uapi; реальная копия drivers/kernelsu/uapi обязательна. При последующих изменениях синхронизируй KSU kernel и uapi заново: редактирование исходного репозитория не обновляет копию. Не копировать .git или ключи. Старые symlink инструкции не использовать для этой проверенной sandbox-сборки.

В workspace:

```sh
cd /path/to/kernel_pixel_6.1
git apply "$RELEASE_DIR/patches/2026100601/0005-local-build-overrides.patch"
```

0005 содержит **host-specific** путь GCC headers `/usr/lib64/gcc/x86_64-alt-linux/13/`, /usr/bin/gcc и жёстко заданные version/signer/package/release. На другой машине корректируй toolchain paths; для своего APK ключа заменяй hash/size согласованно. Опубликованный ключ не распространяется. Release suffix должен соответствовать целевой ОС, его подмена не гарантирует ABI.

Не переносить certificate на новый OTA по имени CN. Проверь DER SHA256 и фактическое embedding. MODULE_SIG_FORCE остаётся включённым. Сборка через tools/bazel и упаковка описаны в ../../INSTALL.md. Последний step — временный boot, не немедленный flash.
