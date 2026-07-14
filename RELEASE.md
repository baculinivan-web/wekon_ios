# Публикация новой версии Wekon в AltStore

Эта инструкция предназначена для будущих релизов iOS-приложения Wekon.

## Что где находится

- Исходный проект: `/Users/a1234/Documents/Wekon for IOS/WekonIOS`
- Hosting-репозиторий: `https://github.com/baculinivan-web/wekon_ios`
- Файл источника AltStore: `altstore.json`
- Публичная ссылка источника: <https://baculinivan-web.github.io/wekon_ios/altstore.json>
- IPA-релизы: раздел [Releases](https://github.com/baculinivan-web/wekon_ios/releases)

В hosting-репозиторий нельзя добавлять исходники приложения, DerivedData, архивы Xcode или сертификаты. В него публикуются только метаданные, изображения и ссылки на IPA-релизы.

## 1. Подготовить новую версию в Xcode

Открой `WekonIOS.xcodeproj` и выбери target `WekonIOS`.

Увеличь оба значения версии:

- `MARKETING_VERSION` — версия для пользователя, например `0.1.1`.
- `CURRENT_PROJECT_VERSION` — целое число сборки, например `2`.

Каждый новый релиз должен иметь новый `version` или `buildVersion`. Обычно увеличиваются оба.

Проверь также `PRODUCT_BUNDLE_IDENTIFIER`: он должен оставаться `ru.wekon.ios`. Минимальная версия iOS текущей сборки — `26.0`; если она изменится, обнови `minOSVersion` в JSON.

## 2. Собрать и подписать IPA

Выполни команды из Terminal:

```bash
cd "/Users/a1234/Documents/Wekon for IOS"

xcodebuild \
  -project WekonIOS/WekonIOS.xcodeproj \
  -scheme WekonIOS \
  -configuration Release \
  -destination 'generic/platform=iOS' \
  -archivePath /tmp/WekonIOS.xcarchive \
  -allowProvisioningUpdates \
  archive
```

После успешного archive экспортируй IPA:

```bash
xcodebuild \
  -exportArchive \
  -archivePath /tmp/WekonIOS.xcarchive \
  -exportPath /tmp/WekonIOS-export \
  -exportOptionsPlist /tmp/WekonExportOptions.plist
```

Если файла `/tmp/WekonExportOptions.plist` ещё нет, создай его со следующим содержимым:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>method</key>
  <string>development</string>
  <key>signingStyle</key>
  <string>automatic</string>
  <key>stripSwiftSymbols</key>
  <true/>
  <key>teamID</key>
  <string>8CCCA37Y8J</string>
</dict>
</plist>
```

Готовый файл должен находиться по адресу `/tmp/WekonIOS-export/WekonIOS.ipa`. Сборка должна быть подписана действующим Apple Development-профилем. AltStore использует IPA для установки и повторно подписывает приложение на устройстве пользователя.

## 3. Проверить метаданные IPA

Проверь размер IPA и значения внутри приложения:

```bash
IPA=/tmp/WekonIOS-export/WekonIOS.ipa
rm -rf /tmp/WekonIPA
mkdir -p /tmp/WekonIPA
unzip -q "$IPA" -d /tmp/WekonIPA

/usr/libexec/PlistBuddy -c 'Print :CFBundleShortVersionString' /tmp/WekonIPA/Payload/WekonIOS.app/Info.plist
/usr/libexec/PlistBuddy -c 'Print :CFBundleVersion' /tmp/WekonIPA/Payload/WekonIOS.app/Info.plist
/usr/libexec/PlistBuddy -c 'Print :CFBundleIdentifier' /tmp/WekonIPA/Payload/WekonIOS.app/Info.plist
stat -f '%z' "$IPA"
shasum -a 256 "$IPA"
```

Запиши размер в байтах: это значение понадобится для поля `size` в `altstore.json`. Значения `version` и `buildVersion` должны точно совпадать с `Info.plist`.

## 4. Подготовить hosting-репозиторий

Клонируй личный репозиторий отдельной рабочей копией:

```bash
rm -rf /tmp/wekon-altstore
/opt/homebrew/bin/gh repo clone baculinivan-web/wekon_ios /tmp/wekon-altstore
cd /tmp/wekon-altstore
mkdir -p screenshots
```

Скопируй новые скриншоты в каталог `screenshots/`. Для iPhone используй портретные изображения, например `1170×2532`:

```bash
cp "/path/to/IMG_3554.png" screenshots/IMG_3554.png
cp "/path/to/IMG_3555.png" screenshots/IMG_3555.png
```

Имена файлов должны быть уникальными или соответствовать тем, которые указаны в JSON.

## 5. Обновить `altstore.json`

В `apps[0].versions` добавь новый объект в начало массива. Пример:

```json
{
  "version": "0.1.1",
  "buildVersion": "2",
  "date": "2026-07-14T12:00:00+03:00",
  "localizedDescription": "Исправления и улучшения.",
  "downloadURL": "https://github.com/baculinivan-web/wekon_ios/releases/download/v0.1.1/WekonIOS.ipa",
  "size": 27000000,
  "minOSVersion": "26.0"
}
```

В поле `size` укажи фактический размер IPA в байтах, а не округлённое значение. Старые версии не удаляй: AltStore использует их как совместимые предыдущие версии.

Обнови список `apps[0].screenshots`, если добавились новые изображения:

```json
"screenshots": [
  {
    "imageURL": "https://baculinivan-web.github.io/wekon_ios/screenshots/IMG_3554.png",
    "width": 1170,
    "height": 2532
  },
  {
    "imageURL": "https://baculinivan-web.github.io/wekon_ios/screenshots/IMG_3555.png",
    "width": 1170,
    "height": 2532
  }
]
```

Не меняй без необходимости `bundleIdentifier`, `appPermissions`, `iconURL` и базовые URL. `appPermissions` должны соответствовать фактическому IPA: если в `Info.plist` появилось новое поле `*UsageDescription`, добавь его в JSON.

Проверь синтаксис:

```bash
python3 -m json.tool altstore.json >/dev/null && echo "JSON OK"
```

## 6. Создать GitHub Release и загрузить IPA

Используй полный путь к GitHub CLI:

```bash
/opt/homebrew/bin/gh auth status

/opt/homebrew/bin/gh release create v0.1.1 \
  /tmp/WekonIOS-export/WekonIOS.ipa \
  --repo baculinivan-web/wekon_ios \
  --title "Wekon 0.1.1" \
  --notes "Описание изменений релиза." \
  --latest
```

Тег релиза должен совпадать с тегом в `downloadURL`. Не переиспользуй уже опубликованный тег: для каждой новой IPA используй новую версию, например `v0.1.1`, `v0.1.2`.

## 7. Опубликовать обновлённый JSON и изображения

После успешной загрузки IPA закоммить JSON и изображения:

```bash
cd /tmp/wekon-altstore
git add altstore.json screenshots
git commit -m "feat: publish Wekon 0.1.1"
git push origin main
```

GitHub Pages автоматически соберёт сайт из ветки `main`. Обычно обновление занимает несколько секунд.

## 8. Проверить публикацию

Проверь JSON, изображения и IPA публичными запросами:

```bash
curl -L --fail --silent --show-error \
  https://baculinivan-web.github.io/wekon_ios/altstore.json \
  | python3 -m json.tool >/dev/null

curl -L --fail --silent --show-error -o /dev/null \
  -w 'IMG_3554: HTTP %{http_code}, bytes %{size_download}\n' \
  https://baculinivan-web.github.io/wekon_ios/screenshots/IMG_3554.png

curl -L --fail --silent --show-error -o /dev/null \
  -w 'IPA: HTTP %{http_code}, bytes %{size_download}\n' \
  https://github.com/baculinivan-web/wekon_ios/releases/download/v0.1.1/WekonIOS.ipa

/opt/homebrew/bin/gh api repos/baculinivan-web/wekon_ios/pages/builds/latest \
  --jq '{status,commit,error}'
```

Ожидаемый результат: JSON проходит проверку, все ссылки возвращают `HTTP 200`, а Pages имеет статус `built`.

## Частые ошибки

- `version` или `buildVersion` не совпадает с IPA — AltStore может отклонить обновление.
- В `downloadURL` указан неправильный тег или имя файла — IPA будет недоступен.
- В JSON указан старый размер IPA — карточка приложения покажет неверный размер.
- JSON отправлен раньше IPA — несколько секунд ссылка может возвращать 404; сначала создавай Release, затем публикуй JSON.
- В репозиторий случайно добавлен исходный проект — используй только `/tmp/wekon-altstore`, а не каталог `WekonIOS`.
- Скриншоты возвращают 404 — дождись завершения Pages build и проверь регистр букв в имени файла.
