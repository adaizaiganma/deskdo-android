# Deskdo Android

> **已經搬到 [adaizaiganma/deskdo](https://github.com/adaizaiganma/deskdo)**（`android/`）。從 1.0.0 開始，電腦版和 Android 版在同一個 repo、同一個 Release，新版 APK 請到 [那邊的 Releases](https://github.com/adaizaiganma/deskdo/releases/latest) 下載。
>
> 這個 repo 保留給 0.7.0–0.10.x 的 App 檢查更新用：它們會在這裡找到 1.0.0，更新之後就改查新的 repo。以下是搬家前的說明。

Deskdo 的 Android 版：待辦清單 App，加上手機主畫面的小工具。資料存在你自己的 Cloudflare 帳號，和[電腦版](https://github.com/adaizaiganma/deskdo)同步，也可以透過 MCP 連接器讓 Claude 等 AI 助理讀寫。

**一般使用者**：到 [Releases](../../releases/latest) 下載 `Deskdo_*_arm64.apk` 安裝。第一次開啟時，設定精靈會帶你登入 Cloudflare；已經在電腦上設定過的話，用同一個帳號登入就會把這支手機加入。

## 結構

| 目錄 | 內容 |
|---|---|
| `src/` | 手機版的介面（Svelte 5） |
| `src-tauri/` | Tauri 2 App：主視窗、偏好設定、APK 更新檢查（`updater.rs`）、和主畫面小工具之間的橋（`mobile.rs`） |
| `src-tauri/gen/android/` | Android 專案；主畫面小工具是 Kotlin（`DeskdoWidget.kt`、`WidgetPlugin.kt`、`WidgetStore.kt`） |
| `upstream/` | git submodule：[adaizaiganma/deskdo](https://github.com/adaizaiganma/deskdo)。用到裡面的 `core/`（和電腦版共用的同步、資料格式、設定精靈）和 `worker/`（設定精靈部署的雲端） |

前端用 `@deskdo/core/...` 匯入共用的程式（npm workspace 指到 `upstream/core`），Rust 依賴 `upstream/core/rust`（crate `deskdo-core`）。

## 開發

```powershell
git clone --recurse-submodules https://github.com/adaizaiganma/deskdo-android
npm ci --prefix upstream/worker
npm ci
npm run build:worker        # 打包 Worker，設定精靈會把它嵌進 App
npm run check               # Svelte／TypeScript 型別檢查
npm test
cd src-tauri; cargo test    # 在電腦上也能編譯（Android 專用的呼叫用 cfg(mobile) 隔開）
npx tauri android dev       # 需要 Android SDK、NDK
```

### 更新共用的程式

共用的程式改在 deskdo repo。推上去之後，這裡更新 submodule：

```powershell
cd upstream; git fetch; git checkout <commit>; cd ..
git add upstream; git commit -m "Update core"
```

## 發布

```powershell
git tag v0.7.0
git push origin v0.7.0
```

`.github/workflows/release.yml` 會建置 APK 並建立 GitHub Release 草稿，確認後按發布。需要在 repo 的 Actions 設定：

- variable `DESKDO_CF_CLIENT_ID`：Cloudflare OAuth client ID
- secrets `ANDROID_KEYSTORE_BASE64`（.jks 轉 base64）、`ANDROID_KEYSTORE_PASSWORD`

版本號在 `package.json`、`src-tauri/Cargo.toml`、`src-tauri/tauri.conf.json` 三處，和電腦版各自獨立。

### 簽名金鑰

APK 用作者的金鑰簽名：`~\.tauri\deskdo-android.jks`（PKCS12，別名 `deskdo`），密碼在 `deskdo-android.jks.password`。**請備份**：換了金鑰，已安裝的 App 就不能直接更新，只能解除安裝再重裝。App 的 identifier 是 `dev.deskdo.app`，不要改。

本機打包：`src-tauri/gen/android/keystore.properties`（不進 repo）寫 `storeFile=` 和 `passwordFile=`，Gradle 就會用它簽 release 版。Windows 沒開「開發人員模式」時 `tauri android build` 建不了 symlink，要自己把 `.so` 複製到 `jniLibs` 再跑 Gradle。

### 更新

App 會檢查這個 repo 的 `releases/latest` 有沒有比較新的 `.apk`，有的話在主畫面提示，按了用瀏覽器下載安裝。0.6.0 以前的版本是到 deskdo repo 檢查，看不到這裡的新版，要手動安裝一次。
