---
title: "Tauriのプラグインの利用方法"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Tauri"]
published: true
---

## はじめに

Tauriのプラグインの利用方法について、プラグインを用いているだけの[簡易なプロジェクト](https://github.com/k-maeda03/tauri-feature-showcase)を用いて解説します。

Tauriの始め方とデバッグ方法については[こちら](https://zenn.dev/maedakou/articles/457e7f9884fbf9)の記事で解説しています。
この記事も学習ログな趣もあるため間違っていたらすみません、指摘頂きたいです。

先ほどのGithubリポジトリをcloneしたら、`npm i && npm run tauri dev`で一通り遊んでみてください。
一応[リリース物](https://github.com/k-maeda03/tauri-feature-showcase/releases/tag/main)もありますが、Windows以外動作確認していないため悪しからず。。。

## プロジェクト構成

```
tauri-app/
├── src/                     # フロントエンド（React）
│   ├── components/          # UIコンポーネント
│   ├── assets/             # 静的リソース
│   └── main.tsx            # エントリーポイント
├── src-tauri/              # バックエンド（Rust）
│   ├── src/
│   │   ├── main.rs        # メインプロセス
│   │   └── lib.rs         # ライブラリ設定
│   ├── Cargo.toml         # Rust依存関係
│   ├── tauri.conf.json    # Tauri設定
│   └── capabilities/      # 権限設定
└── package.json           # Node.js依存関係
```

## Tauri 2.0の新機能：プラグインシステム

### プラグインとは？

Tauri 2.0では、ファイルシステムアクセス、ダイアログ表示、ネットワーク通信などの機能が**プラグイン**として提供されています。
今回はRustに触らずにフロントエンド側の実装だけで、OS機能を触れることをアピールしたいです。

### プラグインの追加方法

公式ドキュメント読めばこの記事は読む必要ないっちゃないです。
色んなプラグインがあります。

https://v2.tauri.app/ja/plugin/

```bash
# プラグインの自動インストール（Tauri CLI推奨）
npm run tauri add fs          # ファイルシステム
npm run tauri add dialog      # ダイアログ
npm run tauri add os          # OS情報
npm run tauri add shell       # シェルコマンド
npm run tauri add clipboard-manager  # クリップボード
npm run tauri add notification       # 通知
npm run tauri add http               # HTTP通信
```

このコマンドは自動的に以下を実行します：
- `package.json`にJavaScript/TypeScriptパッケージを追加
- `src-tauri/Cargo.toml`にRustクレートを追加  
- `capabilities`設定を更新

このときcapabilitiesではdefaultが各プラグインに設定されます。  
`shell`や`clipboard-manager`、`http`などはdefaultだと何もできないに等しいので、ドキュメントを参考に適切に設定する必要があります。  
この辺もドキュメントの各プラグインのページ読めば分かります。

## 実践編：各プラグインの使用方法

### 1. ファイルシステム操作（@tauri-apps/plugin-fs）

ファイルの読み書き、ディレクトリ操作を行えます。

```typescript
import { open, save } from "@tauri-apps/plugin-dialog";
import { readTextFile, writeTextFile, readDir } from "@tauri-apps/plugin-fs";

// ファイル選択・読み取り
async function openFile() {
    const selected = await open({ 
        multiple: false,
        filters: [{ name: 'テキスト', extensions: ['txt', 'md'] }]
    });
    
    if (typeof selected === "string") {
        const content = await readTextFile(selected);
        console.log("ファイル内容:", content);
    }
}

// ファイル保存
async function saveFile(content: string) {
    const selected = await save({ defaultPath: "note.txt" });
    if (typeof selected === "string") {
        await writeTextFile(selected, content);
        console.log("ファイルを保存しました");
    }
}

// ディレクトリ一覧
async function listDirectory() {
    const selected = await open({ directory: true });
    if (typeof selected === "string") {
        const entries = await readDir(selected);
        entries.forEach(entry => console.log(entry.name));
    }
}
```

### 2. ダイアログ表示（@tauri-apps/plugin-dialog）

各種システムダイアログを表示できます。

```typescript
import { message, ask, confirm, open, save } from "@tauri-apps/plugin-dialog";

// 情報メッセージ
await message("操作が完了しました", { 
    title: "情報", 
    kind: "info" 
});

// Yes/No質問
const answer = await ask("続行しますか？", { 
    title: "確認", 
    kind: "warning" 
});
console.log(answer ? "はい" : "いいえ");

// OK/Cancel確認
const confirmed = await confirm("この操作を実行しますか？", { 
    title: "確認",
    kind: "info" 
});
console.log(confirmed ? "実行" : "キャンセル");
```

### 3. システム情報取得（@tauri-apps/plugin-os）

OS情報やシステム詳細を取得できます。

```typescript
import * as os from "@tauri-apps/plugin-os";

// システム情報を取得
const systemInfo = {
    platform: await os.platform(),    // "windows" | "macos" | "linux"
    arch: await os.arch(),            // "x86_64" | "aarch64" など
    version: await os.version(),      // OSバージョン
    type: await os.type()            // OSタイプ
};

console.log("システム情報:", systemInfo);
```

### 4. シェルコマンド実行（@tauri-apps/plugin-shell）

安全にシェルコマンドを実行できます。

```typescript
import { Command } from "@tauri-apps/plugin-shell";

// コマンド実行
async function runCommand(command: string) {
    const cmd = Command.create("echo", ["Hello, Tauri!"]);
    
    try {
        const output = await cmd.execute();
        console.log("標準出力:", output.stdout);
        console.log("エラー出力:", output.stderr);
    } catch (error) {
        console.error("コマンド実行エラー:", error);
    }
}
```

### 5. クリップボード操作（@tauri-apps/plugin-clipboard-manager）

クリップボードの読み書きができます。

```typescript
import { writeText, readText } from "@tauri-apps/plugin-clipboard-manager";

// クリップボードに書き込み
async function copyToClipboard(text: string) {
    try {
        await writeText(text);
        console.log("クリップボードにコピーしました");
    } catch (error) {
        console.error("コピーに失敗しました:", error);
    }
}

// クリップボードから読み取り
async function pasteFromClipboard() {
    try {
        const text = await readText();
        console.log("クリップボードの内容:", text);
        return text;
    } catch (error) {
        console.error("貼り付けに失敗しました:", error);
        return "";
    }
}
```

### 6. システム通知（@tauri-apps/plugin-notification）

デスクトップ通知を送信できます。

```typescript
import { 
    isPermissionGranted, 
    requestPermission, 
    sendNotification 
} from '@tauri-apps/plugin-notification';

async function showNotification(title: string, body: string) {
    // 通知許可の確認
    let permissionGranted = await isPermissionGranted();
    
    if (!permissionGranted) {
        const permission = await requestPermission();
        permissionGranted = permission === 'granted';
    }
    
    if (permissionGranted) {
        sendNotification({ title, body });
    } else {
        console.log("通知の許可が得られませんでした");
    }
}
```

### 7. HTTP通信（@tauri-apps/plugin-http）

安全にHTTP APIを呼び出せます。

```typescript
import { fetch } from "@tauri-apps/plugin-http";

// API呼び出し
async function callApi() {
    try {
        const response = await fetch("https://api.example.com/data");
        const data = await response.json();
        console.log("APIレスポンス:", data);
        return data;
    } catch (error) {
        console.error("API呼び出しエラー:", error);
        return null;
    }
}
```

## セキュリティ：Capabilitiesの設定

### 権限設定の重要性

Tauriはセキュリティファーストの設計により、デフォルトでは最小限の権限のみが許可されています。`src-tauri/capabilities/default.json`で必要な権限を明示的に設定する必要があります。

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "default",
  "description": "メインウィンドウの権限設定",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "fs:default",                    // ファイルシステム
    "dialog:default",               // ダイアログ
    "os:default",                   // OS情報
    "clipboard-manager:default",     // クリップボード
    "notification:default",         // 通知
    "http:default",                 // HTTP通信
    {
      "identifier": "shell:allow-execute",
      "allow": [
        {
          "name": "echo",           // echoコマンドのみ許可
          "args": true
        }
      ]
    }
  ]
}
```

### セキュリティのベストプラクティス

1. **最小権限の原則**: 必要最小限の権限のみを付与
2. **明示的な許可**: 各機能を個別に許可設定
3. **コマンド制限**: シェル実行は特定のコマンドのみ許可
4. **入力検証**: ユーザー入力は必ず検証してからAPIに渡す

## まとめ

Tauri 2.0は、Web技術でネイティブアプリを作成するための強力で安全なフレームワークです。プラグインシステムにより必要な機能のみを追加でき、セキュリティと軽量性を両立できます。

このリポジトリのサンプルアプリケーションを参考に、ぜひ独自のTauriアプリケーションを作成してみてください！

---

📝 **この記事が役に立ったら、GitHubでスターをつけていただけると嬉しいです！**

🔗 **サンプルリポジトリ**: [tauri-feature-showcase](https://github.com/k-maeda03/tauri-feature-showcase)