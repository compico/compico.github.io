---
title: "Решение проблем"
description: "Что проверить, если оверлей или OBS работают некорректно."
weight: 5
---

# Решение проблем

## Оверлей не отображается в OBS

1. Открой в браузере http://127.0.0.1:8000/overlay?debug=1 или добавь ?debug=1 к URL источника Browser Source в OBS.
2. Проверь статус, отображаемый в оверлее.
3. В OverlayComposer открой Settings → Open log folder и посмотри последние записи журнала.
4. Убедись, что URL Browser Source и порт совпадают с настройками приложения.

## Виджеты тормозят во время игры

В некоторых конфигурациях Windows аппаратное ускорение Browser Source в OBS может конкурировать с игрой за ресурсы GPU. Для проверки открой OBS → Settings → Advanced, отключи Browser Source Hardware Acceleration и перезапусти OBS.

## Нужна помощь?

Создай issue в [репозитории OverlayComposer](https://github.com/compico/overlaycomposer/issues). Перед отправкой журнала удали из него токены, приватные URL и другие конфиденциальные данные.
