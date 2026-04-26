# Microsoft Agent Framework for Python — 日本語環境向け

Python プロジェクトで Microsoft Agent Framework を使うときの参照。

## Authoritative sources

- Docs: https://learn.microsoft.com/agent-framework/
- Repository: https://github.com/microsoft/agent-framework/tree/main/python
- Samples: https://github.com/microsoft/agent-framework/tree/main/python/samples

## Package

最新 docs と package version を確認してから追加する。

```powershell
pip install agent-framework
```

プロジェクトに `pyproject.toml`、`requirements.txt`、`uv.lock`、`poetry.lock` がある場合は既存管理方法に合わせる。

## Python 方針

- async / await を基本にする。
- 型ヒントを付ける。
- prompt、tool、workflow、configuration を分離する。
- secret は環境変数または secret store から取得する。
- structured output には dataclass / Pydantic などを検討する。
- 最新 samples を確認してから API を使う。

## Configuration

環境変数例:

```powershell
$env:AZURE_OPENAI_ENDPOINT = "https://example.openai.azure.com/"
$env:AZURE_OPENAI_DEPLOYMENT = "gpt-4.1"
```

secret を `.env` に置く場合は `.gitignore` を確認し、リポジトリへ含めない。

## Testing

- 単体テストで実 model を呼ばない。
- provider client を fake / stub に差し替える。
- tool の validation と authorization をテストする。
- workflow routing を deterministic にテストする。

## 日本語環境

- prompt / instructions は日本語でよい。
- API 名、JSON field、tool name は英語を維持する。
- UTF-8 前提でファイルを扱う。
- 日本語検索や要約では表記ゆれを考慮する。

## Review checklist

- [ ] async API になっている
- [ ] 型ヒントがある
- [ ] secret をコードに直書きしていない
- [ ] prompt / tools / workflow が分離されている
- [ ] 実 model に依存しないテストがある
- [ ] 最新 Python samples と整合している
