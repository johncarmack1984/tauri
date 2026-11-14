---
"tauri": major:breaking
---

The `specta` feature no longer implements `specta::function::FunctionArg` for `State`, `AppHandle`, `Window`, `Webview` and `WebviewWindow` or enables specta's `function` feature: describe a command with `tauri::command_metadata!` and the `specta::Type` of its arguments instead of `#[specta::specta]`.
