# Лабораторная работа №4. Облачные хранилища. Amazon S3

**ФИО:** Gorbatiuc Evelina
**Группа:** IA2403
**Специальность:** Informatică aplicată
**Уровень:** базовый (задания 1–6)
**Регион:** eu-central-1 (Frankfurt)
**Бакет:** `timecapsule-dev-uploads-evelina`

---

## Задание 1. Бакет и объекты в консоли

Создала бакет `timecapsule-dev-uploads-evelina`: General purpose, ACLs disabled, Block all public access, шифрование SSE-S3, тег `Lab = lab4`. Загрузила картинку и PDF в `uploads/`, текстовый файл в `backups/`.

Свойства картинки:

| Параметр | Значение |
| --- | --- |
| Key | `uploads/Снимок экрана 2026-10-31 101315.png` |
| Type | `image/png` |
| ETag | `20ae04379feae3241434e96f481013cc` |
| Object URL | `https://timecapsule-dev-uploads-evelina.s3.eu-central-1.amazonaws.com/uploads/...` |

**Скриншоты:**

![01-bucket-created](screenshots/01-bucket-created.png)

![01-upload-png](screenshots/01-upload-png.png)

![01-upload-backups](screenshots/01-upload-backups.png)

![01-upload-pdf](screenshots/01-upload-pdf.png)

![01-object-overview](screenshots/01-object-overview.png)

![01-object-url-denied](screenshots/01-object-url-denied.png)


**Какой ключ у картинки?** `uploads/Снимок экрана 2026-10-31 101315.png`. Это вся строка целиком, папок в S3 нет, `uploads/` является частью имени ключа.

**Почему Object URL не открывается, хотя я владелец?** Бакет закрыт для публичного доступа, а браузер отправляет анонимный запрос без подписи. Права владельца действуют только для подписанных запросов.

**Что создала кнопка Create folder?** Пустой объект нулевого размера с ключом `uploads/`. Консоль показывает его как папку.

---

## Задание 2. Объекты из командной строки

Выполнила в CloudShell: `aws s3 cp`, `aws s3 sync`, три варианта `aws s3 ls`, `head-object` для картинки и `report.txt`, скачивание `photo.txt` и удаление `report-2.txt`.

**Скриншоты:**

![02-cli-upload-sync](screenshots/02-cli-upload-sync.png)

![02-ls-levels](screenshots/02-ls-levels.png)

![02-head-object](screenshots/02-head-object.png)

![02-cp-rm](screenshots/02-cp-rm.png)


**Почему первая команда `ls` показывает `PRE`, а третья полные ключи?** Без `--recursive` S3 группирует ключи по разделителю `/` и возвращает общие префиксы (`PRE uploads/`). С `--recursive` разделитель не используется, выводятся все ключи.

**Какой Content-Type получил `report.txt`?** `text/plain`. AWS CLI определил его по расширению `.txt`.

---

## Задание 3. Политика бакета: только HTTPS

Добавила политику с `Effect: Deny`, `Principal: "*"`, `Action: s3:*`, ресурсами бакета и `bucket/*`, условием `aws:SecureTransport = false`.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::timecapsule-dev-uploads-evelina",
        "arn:aws:s3:::timecapsule-dev-uploads-evelina/*"
      ],
      "Condition": { "Bool": { "aws:SecureTransport": "false" } }
    }
  ]
}
```

**Скриншоты:**

![03-bucket-policy](screenshots/03-bucket-policy.png)

![03-https-check](screenshots/03-https-check.png)


**Почему вторая команда получила AccessDenied, хотя у меня права администратора?** Права администратора это Allow, а явный запрет (Deny) в AWS всегда сильнее любого разрешения.

**Почему Block Public Access не помешал сохранить `"Principal": "*"`?** Он блокирует политики, которые открывают доступ всем. Моя политика запрещает, поэтому бакет не становится публичным.

---

## Задание 4. Версионирование и защита от удаления

Включила версионирование. Файлы, загруженные до этого, имеют Version ID `null`. Загрузила `notes.txt` (v1), затем (v2). Удалила объект при выключенном Show versions: появился Delete marker. Удалила маркер, и файл вернулся с текстом v2. Удалила версию v2, и текущей стала v1.

**Скриншоты:**

![04-versioning-enable](screenshots/04-versioning-enable.png)

![04-versions-null](screenshots/04-versions-null.png)

![04-two-versions](screenshots/04-two-versions.png)

![04-delete-marker](screenshots/04-delete-marker.png)

![04-restored](screenshots/04-restored.png)

![04-rollback-v1](screenshots/04-rollback-v1.png)


**Чем удаление в шаге 3 отличается от удаления в шаге 5?** В шаге 3 S3 только ставит Delete marker, данные остаются. В шаге 5 удалена конкретная версия, это окончательно.

**Какое можно отменить?** Только удаление в шаге 3: достаточно удалить маркер.

**Какой способ защищает от случайного удаления, а какой нужно запрещать приложению?** Обычное удаление с маркером защищает от случайной потери. Удаление конкретной версии нужно запрещать приложению, иначе оно сможет уничтожить историю без возможности восстановления.

---

## Задание 5. Правило жизненного цикла

Правило `timecapsule-dev-noncurrent`, область: префикс `uploads/`.

- Неактуальные версии переходят в Standard-IA через 30 дней.
- Неактуальные версии удаляются через 90 дней, кроме двух самых новых.
- Незавершённые загрузки частями удаляются через 7 дней.

**Скриншоты:**

![05-lifecycle-review](screenshots/05-lifecycle-review.png)

![05-lifecycle-list](screenshots/05-lifecycle-list.png)


**Как поведёт себя правило для файла, перезаписанного 10 раз за неделю?** Будет одна текущая версия и девять неактуальных. Через 30 дней после замены каждая неактуальная версия может перейти в Standard-IA. Через 90 дней все неактуальные версии, кроме двух самых новых, будут удалены. Текущая версия не удаляется.

**Почему тестовые файлы в Standard-IA не попадут?** Минимальный размер объекта для перехода 128 КБ, наши файлы по несколько байт.

**Что такое незавершённая загрузка частями и почему за неё платят?** Большой файл загружается частями. Если загрузка прервалась, загруженные части остаются в бакете и не видны в обычном списке, но занимают место и оплачиваются как хранение. Правило удаляет их через 7 дней.

---

## Задание 6. Подписанные URL

1. Ссылка из консоли на 5 минут для картинки открылась в приватном окне.
2. Ссылка из CLI: `aws s3 presign s3://$BUCKET/uploads/photo.txt --expires-in 120`. В приватном окне открылось содержимое `photo placeholder`.
3. Заменила `photo.txt` на `notes.txt` в адресе: S3 вернул `SignatureDoesNotMatch`.
4. Открыла исходную ссылку после истечения срока: S3 вернул `AccessDenied`, `Request has expired`.

**Скриншоты:**

![06-console-presigned](screenshots/06-console-presigned.png)

![06-cli-presign](screenshots/06-cli-presign.png)

![06-photo-placeholder](screenshots/06-photo-placeholder.png)

![06-signature-mismatch](screenshots/06-signature-mismatch.png)

![06-expired](screenshots/06-expired.png)

Подписи в ссылках размыты, срок действия ссылок истёк.


**Какие ошибки вернул S3?** В шаге 3 `SignatureDoesNotMatch`, в шаге 4 `AccessDenied: Request has expired`.

**Почему нельзя получить другой файл, поменяв ключ?** Ключ объекта входит в подписываемые данные. Подпись выдана для конкретного ключа и при его замене не совпадает.

**Обращалась ли `aws s3 presign` к S3?** Нет. Подпись вычисляется локально по учётным данным, обращение к S3 происходит только при открытии ссылки.

---

## Завершение

Временные файлы в CloudShell удалены (`rm -rf lab4`). Бакет не удалён: он нужен в следующих работах.
