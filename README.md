# Frontend (Vue + Tauri)

Интерфейс личного цифрового трекера:
- форма создания цели;
- карточки целей с прогресс-баром;
- чек-ины прогресса;
- сезонный дашборд итогов.

Данные сохраняются локально в `localStorage` (backend не требуется).

## Web
```bash
npm install
npm run dev
```

Сборка web-версии:
```bash
npm run build
```

## Tauri
В проекте уже настроен `src-tauri`.

Запуск Tauri в режиме разработки:
```bash
npm run tauri:dev
```

Сборка desktop-приложения:
```bash
npm run tauri:build
```

### Требования для Windows
- Rust + Cargo через `rustup` (`https://rustup.rs/`);
- Visual Studio Build Tools 2022 с компонентами MSVC и Windows SDK;
- WebView2 Runtime (обычно уже установлен).
