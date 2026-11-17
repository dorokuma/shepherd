# AGENTS.md

Shepherd は Herdr 管理の coding agent から agent snapshot、`agent.*` event、orchestrator notification を作る TypeScript daemon / CLI です。まず `README.md` で使い方を確認し、仕様判断が必要なときだけ `docs/plans/` を見てください。

## よく使うコマンド

- `mise install`: `mise.toml` の Node.js / pnpm を入れる。
- `pnpm install`: 依存関係を入れる。
- `pnpm check`: typecheck、test、Biome、Drizzle、Pi package、Herdr plugin の検証をまとめて実行する。
- `pnpm test`: Vitest を一回実行する。
- `pnpm test:watch`: Vitest の watch。
- `pnpm build`: 古い`dist`を削除してTypeScriptを出力し、`tsc-alias`でimport aliasを解決する。
- `pnpm package:check`: root npm packageをbuildし、tarballのfile allowlistを検証する。
- `pnpm lint:fix`: Biome の lint/import/format fix を適用する。
- `pnpm db:generate`: `src/db/schema.ts` から SQL migration を生成する。
- `SHEPHERD_HOME=/tmp/shepherd pnpm db:migrate`: 指定した Shepherd home の SQLite DB に migration を適用する。

## 検証手順

- 実装変更後は `pnpm check` を通す。
- CLI entrypoint、`dist`のimport解決、package内容に関わる変更では`pnpm build`と`pnpm package:check`も通す。
- DB schema を変えたら `pnpm db:generate` を先に実行し、生成 SQL を確認してから migrate を見る。
- Node / pnpm の PATH が古い環境では、検証コマンドの前に次を付ける。

```bash
PATH="$HOME/.local/share/mise/installs/node/24.18.0/bin:$HOME/.local/share/mise/installs/pnpm/11.9.0/bin:$PATH"
```

## 重要パス

- `src/observability/`: agent contract、cached agent context、agent index、orchestrator service。
- `src/daemon/`: daemon JSON Lines RPC、process manager、service startup。
- `src/cli/`: `shepherd` CLI entrypoint。
- `src/config/`: runtime config schema と path/env 解決。
- `src/db/`: SQLite connection、Drizzle schema、migration runner、observability store。
- `src/herdr/`: Herdr socket client、managed session client、session snapshot、workspace resolver。
- `src/shared/`: JSON Lines framing などの共有 utility。
- `packages/shepherd-pi/`: npmで公開するPi extension package。
- `packages/shepherd-herdr-plugin/`: GitHub経由で配布するprivate Herdr integration。npmには公開しない。
- `test/unit/`: pure logic / contract tests。
- `test/integration/`: SQLite / JSON Lines RPC など実体を使う tests。
- `docs/plans/`: active plan。完了済み plan は `docs/plans/archived/` に置く。

## コーディング方針

- チャットでの応答は日本語。
- TypeScript は ESM + `NodeNext`。`src` 配下は `@/*` import alias を使う。
- Runtime schema は TypeBox/Ajv、DB schema は Drizzle に寄せる。
- 変更は既存の層に合わせる。transport、persistence、observability rules、runtime extension の責務を混ぜない。
- Markdown docs は Biome gate の対象外。変更した docs はリンクとコマンドを目視で確認する。
- README の利用例や詳細設計を AGENTS.md に重複させない。

## Plan / docs 運用

- npmとGitHubのrelease手順は`docs/releasing.md`を正とする。公開するnpm packageはrootと`packages/shepherd-pi`の2つだけ。
- Active plan は `docs/plans/` 配下に置く。完了済み plan は `docs/plans/archived/` 配下に移す。
- 大きな plan は親 plan と子 plan に分ける。親は目的、方針、進捗、子 plan link に絞る。
- 子 plan ディレクトリ名は親 plan ファイル名から `.md` を除いた名前と一致させる。
- plan には `Status`、`Progress`、`Next steps` を置く。
- plan 更新時は親子リンク、ディレクトリ名、README / AGENTS からの参照を確認する。
- 完了済み plan の archive は docs-only commit に分ける。

## 注意

- `node_modules/`, `dist/`, `*.sqlite` は commit しない。
- `pnpm-workspace.yaml` は pnpm 11 の `allowBuilds` 用。workspace 化の意図で編集しない。
- ユーザーや別プロセスの未コミット変更を戻さない。
## Release Policy / 发布规范

本节是本仓库发版的强制规则。仓库已有的更具体版本、构建或发布渠道规则优先，但不得降低以下底线。

1. **版本与变更记录**：遵循 Semantic Versioning，正式版本使用 `vX.Y.Z`。破坏性变更升 MAJOR，新功能升 MINOR，兼容修复升 PATCH。任何用户可见行为、运行时/API/包接口、依赖、安全性或分发产物变化，都必须在同一发版提交中更新版本元数据并写入 CHANGELOG/Release Notes；纯文档且不产生可分发产物时可免升版，但须在规则或 Release Notes 中明确说明。
2. **发布门禁**：发版前工作树必须干净，版本号、CHANGELOG 与发布说明必须一致；必须运行仓库规定的测试、构建、lint、类型检查及必要的安全检查，且全部通过。若仓库提供 `release-check`、`make release-check`、版本检查脚本或同等门禁，必须原样通过，不得绕过。
3. **发布顺序**：先提交并推送发布提交到目标分支 → 在该发布提交上创建 Tag（默认使用带注解的 `git tag -a vX.Y.Z -m "vX.Y.Z"`；仅当现有发布工具明确要求轻量 Tag 时可例外）→ 推送 Tag → 创建**非草稿** GitHub Release → 再发布 npm、PyPI、容器镜像等关联分发渠道。不得只推分支或只打 Tag 就宣称发版完成。
4. **GitHub Release 产物**：标题使用 `vX.Y.Z`，正文必须包含实质性变更摘要、升级/破坏性变更说明及验证结果；二进制、安装包、插件包等适用产物必须作为 Release assets 上传，并提供校验和（如适用）。**Tag 本身不等于完整发版，必须同时存在可访问的 GitHub Release。**
5. **发布后验收**：至少执行并确认 `git ls-remote --tags origin refs/tags/vX.Y.Z`、Tag 指向发布提交、`gh release view vX.Y.Z` 可访问且不是 draft；若发布到包仓库/镜像仓库，还必须核验对应版本、摘要或安装链接可用。
6. **不可变性**：已发布 Tag 不得复用、移动或删除，不得 force push 发布分支或 Tag，不得从未提交、未经测试或未通过门禁的树发布。发布后发现问题须通过新的 patch 版本修复，禁止覆盖历史 Release。
7. **例外必须显式化**：归档、已停用、被替代、仅文档或无可分发产物的仓库，必须在项目规则中明确写出“不发新版本”及历史 Tag/Release 的处置方式；例外不能用于绕过上述发布闭环。

### Repository-specific release contract

- 本仓具体权威流程见 `docs/releasing.md`；npm 与 GitHub Release 必须按该文件同步完成。
- - **发布边界**：本仓当前镜像上游 `ryonakae/shepherd` 的 Tag；正式 npm 包与 GitHub Release 以上游发布为权威来源，本 Fork 不重复发布同一 Tag。若形成独立发行线，必须使用不冲突的版本序列并完成完整 Release/产物/registry 验收。
