# Проверенные артефакты 2026100601

Только Pixel8 shiba / GrapheneOS2026100601 / SPL2026-10-05. Не использовать после OTA или на другом устройстве. SHA256SUMS охватывает все пять файлов:

```sh
sha256sum -c SHA256SUMS
```

- `boot-ksu341.img`: встроенный KSU3.4.1/33333, 64MiB, header4, matching module cert, MODULE_SIG_FORCE. SHA256 9e60f21b52ccc0426a28d0b44248cfcd572f581795280001c9b072197978bbc8. Это точный образ, временно проверенный и записанный только в boot_a двух телефонов.
- `Manager-prctl341-with-ksud.apk`: **исправленный полноценный** arm64 APK com.rifsxd.ksunext.prctl341, включает patched libksud.so. SHA256 81e4b8b67b489b948674a6a29068dfbac41ba6bf217b09aa1747aff1f14edc03. Cert SHA256 b4eb1c14210fff0e1d8102e0f54a9cd1dfd674050dc3a520ff88dc8c048b0da3, DER744 байта. Private key не опубликован.
- `stock-boot.img`: rollback reference из официального подписанного архива этой версии, не raw snapshot телефона.
- `stock-init_boot.img`: штатный reference этой версии; восстанавливать **только если раздел менялся**. Новая KSU установка его не трогает.
- `module-signing-cert.pem`: публичный модульный сертификат, извлечённый из matching stock kernel. DER SHA256 9ca70bf606dcf3c61b91340d63cdfdb709168a0ad9a6f6ee723181bdef4af9cd. Это не APK cert и не private key.

AVB Algorithm NONE: проверен hash/footer, не официальная подпись. Bootloader остаётся unlocked. Неполный ранний APK и кандидаты с сертификатом старого выпуска здесь не опубликованы. Полные личные bugreports не включены.

Инструкция ../../INSTALL.md. Бинарники в родительском artifacts/ относятся к **старому 2026100201**, не путать их с этим каталогом.
