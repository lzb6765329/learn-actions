GitVersion 是一个强大的工具，用于基于 Git 历史自动计算语义版本号（SemVer），它支持通过配置文件（GitVersion.yml）自定义规则，包括版本递增逻辑。你提到的 mainline 模式（即 mode: Mainline）是适合 trunk-based 开发的工作流，其他模式如 ContinuousDelivery（在非标签 commit 上添加预发布标签，如 -beta.1，适合持续交付但不立即部署）、ContinuousDeployment（添加 -ci.1 等标签，适合每个 commit 都可能部署）和 ManualDeployment（较少用，强调手动控制发布）可以根据需求选择。但基于你的描述，你想避免 main 分支每次合并 PR 或触发 workflow 时自动递增版本号（patch 或其他），而只通过提交消息（commit message）来判断是否递增，这完全可行。

下面我一步步解释如何实现，特别是针对管理 NuGet 包版本。核心是通过设置 main 分支的 increment: None 来禁用自动递增，然后依赖 commit message 中的 +semver 指令来手动控制递增。这样，只有当你明确在 commit message（或 PR 合并消息）中指定 +semver: major/minor/patch 时，版本才会递增；否则，版本保持不变（即使 main 合并了 PR）。这适合你的场景：有些 PR 不影响 NuGet 包版本，就不用递增。

### 1. **安装和基本设置 GitVersion**
- 如果还没安装，在你的项目中通过 NuGet 添加 GitVersion.Tool（dotnet tool install --global GitVersion.Tool），或在 CI（如 GitHub Actions）中通过 workflow 安装。
- 在仓库根目录创建 GitVersion.yml 文件来配置规则。
- 你可以预设版本号通过：
  - Git tag（如 git tag 1.0.0）来锚定起始版本，GitVersion 会从那里开始计算。
  - 或在 GitVersion.yml 中设置 next-version: 1.0.0 来强制起始版本。

### 2. **配置版本递增规则**
默认下，Mainline 模式会在 main 分支的每个合并 commit 上自动递增 patch（如果有变化）。要改为只基于 commit message 控制：
- 设置 mode: Mainline（适合你的 trunk-based 开发，不添加预发布标签，除非你手动配置）。
- 对于 main 分支，设置 increment: None（禁用自动递增）。
- 保持 commit-message-incrementing: Enabled（默认值），允许通过 commit message 控制。
- 可选：设置 prevent-increment.of-merged-branch: true（默认 true），避免从合并的分支继承自动增量（只看合并消息本身）。
- 如果你想只在 PR 合并消息（merge message）中应用 +semver（忽略 PR 内单个 commit），可以设置 commit-message-incrementing: MergeMessageOnly。

示例 GitVersion.yml（放在仓库根目录）：
```yaml
mode: Mainline  # 或 ContinuousDelivery，如果你想在非发布 commit 上添加预发布标签如 -beta.1
next-version: 1.0.0  # 可选：如果想从特定版本开始计算，删除此行则从 Git 历史自动推导
commit-message-incrementing: Enabled  # 允许 commit message 控制递增
branches:
  main:
    increment: None  # 关键：禁用自动递增，只依赖 +semver
    prevent-increment:
      of-merged-branch: true  # 避免从 feature 分支继承增量
    track-merge-message: true  # 考虑合并消息（PR 标题/描述）
    regex: ^main$  # 匹配 main 分支（如果你用 master，改为 ^master$）
    is-main-branch: true
# 可选：其他分支配置，如 feature 分支默认递增 minor
  feature:
    increment: Minor
    regex: ^features?[/-]
```

- **如何测试配置**：运行 `gitversion` 命令（或 `dotnet gitversion`）在本地查看计算出的版本。输出会包括 FullSemVer、MajorMinorPatch 等变量。

### 3. **通过提交消息控制版本递增**
- 在 commit message（或 PR 的 squash/merge 消息）中添加 +semver 指令来指定递增类型。
- 语法：+semver: [major|minor|patch|none]
  - +semver: major（或 breaking）：递增 major（如 1.2.3 -> 2.0.0），用于破坏性变化。
  - +semver: minor（或 feature）：递增 minor（如 1.2.3 -> 1.3.0），用于新功能。
  - +semver: patch（或 fix）：递增 patch（如 1.2.3 -> 1.2.4），用于 bug 修复。
  - +semver: none（或 skip）：明确跳过递增（版本不变），适合不影响 NuGet 的 PR。
- 示例 commit message：
  ```
  Fix bug in login module +semver: patch
  ```
  - 这会递增 patch。
- 如果消息中没有 +semver，且 increment: None，则版本不变（即使合并到 main）。
- 注意：指令可以放在消息的任何位置，但最好放末尾。GitVersion 会扫描所有相关 commit（包括合并链）。
- 如果用 Conventional Commits（推荐），GitVersion 也能自动映射：如 "feat: new api" -> minor，"fix: bug" -> patch，"BREAKING CHANGE: refactor" -> major。但你仍可叠加 +semver 来覆盖。

### 4. **整合到 NuGet 包管理**
你当前用 nuspec 文件固定版本，通过 PS1 脚本生成包。这很容易和 GitVersion 整合：让脚本调用 GitVersion 计算动态版本，然后注入到 nuget pack 中。这样，避免手动修改 nuspec 的版本。

- **修改你的 PS1 脚本**：
  - 调用 GitVersion 获取版本变量。
  - 示例脚本片段（假设你有 GitVersion.Tool 安装）：
    ```powershell
    # 运行 GitVersion 并捕获输出（JSON 格式）
    $gitVersionOutput = dotnet gitversion /output json | ConvertFrom-Json

    # 获取计算出的版本（如 1.2.3）
    $nugetVersion = $gitVersionOutput.MajorMinorPatch  # 或 FullSemVer 如果想带预发布标签

    # 如果是预发布（如 CI build），可以添加后缀
    if ($gitVersionOutput.PreReleaseTag -ne "") {
        $nugetVersion = "$nugetVersion-$($gitVersionOutput.PreReleaseTag)"
    }

    # 更新 nuspec 文件的版本（用 XML 处理）
    [xml]$nuspec = Get-Content "YourProject.nuspec"
    $nuspec.package.metadata.version = $nugetVersion
    $nuspec.Save("YourProject.nuspec")

    # 生成 NuGet 包
    nuget pack YourProject.nuspec -Version $nugetVersion -OutputDirectory .\packages
    ```
  - 如果用 dotnet CLI 构建（如 .csproj），更好：用 `dotnet pack --version-suffix $suffix` 并设置 AssemblyVersion 通过 GitVersion 的 /updateassemblyinfo。

- **在 CI Workflow（如 GitHub Actions）中集成**：
  - 只在 main 分支触发时运行，但由于版本只在有 +semver 时递增，多次触发也不会无谓递增。
  - 示例 workflow 片段：
    ```yaml
    jobs:
      build:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4
            with: { fetch-depth: 0 }  # 需要完整 Git 历史
          - name: Install GitVersion
            uses: gittools/actions/gitversion/setup@v1
            with: { versionSpec: '5.x' }
          - name: Execute GitVersion
            id: gitversion
            uses: gittools/actions/gitversion/execute@v1
          - name: Build NuGet
            run: |
              echo "Version: ${{ steps.gitversion.outputs.majorMinorPatch }}"
              # 调用你的 PS1 或 dotnet pack -p:PackageVersion=${{ steps.gitversion.outputs.semVer }}
    ```
  - 如果版本没变，NuGet 包版本也不会变（你可以检查是否发布新包）。

### 5. **注意事项和最佳实践**
- **如果版本没递增**：这是预期的——只有 +semver 时才变。测试时，用本地 git commit 添加消息查看。
- **预发布标签**：如果用 ContinuousDelivery 模式，main 上未标签的 commit 会自动加 -beta.1 等（基于 build-metadata）。Mainline 模式避免这个，除非配置。
- **回滚或固定**：用 git tag 固定发布版本（如 1.2.3），后续递增从那里开始。
- **Conventional Commits**：强烈推荐结合使用（如 Angular 风格），GitVersion 会自动处理递增，而 +semver 可作为覆盖。
- **调试**：运行 `gitversion /diag` 查看详细计算逻辑。如果配置不对，版本可能意外不变或递增。
- 如果你的仓库有多个 NuGet 包，考虑 per-project 配置或 monorepo 支持。

这个设置应该完美匹配你的需求：灵活控制 NuGet 版本，只在需要时递增，而不依赖每次 main 变化。如果你有具体仓库结构或出错，分享更多细节我可以细调。

以下是针对你的需求，使用 GitVersion 配置文件的示例（GitVersion.yml）。你的目标是：只在合并（merge）到 main 分支时，检查合并提交的消息是否包含 SemVer 相关的关键词（如 +semver: major/minor/patch，或者 Conventional Commits 风格如 feat:、fix: 等）来决定版本号增量。分支上的单个提交不会触发增量，只有 main 上的合并提交消息决定。

GitVersion 通过设置 `commit-message-incrementing: MergeMessageOnly` 来实现只检查合并消息。同时，使用 `Mainline` 模式，确保版本计算基于 main 分支的历史，并启用 `track-merge-message: true` 来解析合并消息。

### GitVersion.yml 示例配置
```yaml
mode: Mainline  # 使用 Mainline 模式，只考虑 main 分支上的提交历史（包括合并提交），适合 GitHub Flow 风格的工作流

commit-message-incrementing: MergeMessageOnly  # 关键设置：只通过合并提交的消息来决定版本增量，忽略 PR 内单个提交的消息

branches:
  main:
    regex: ^main$  # 匹配 main 分支
    mode: ContinuousDeployment  # main 分支始终可部署，版本号会包含预发布标签（如 -ci.1）
    increment: Patch  # 默认增量为 patch（如果消息无指定）
    prevent-increment:
      of-merged-branch: true  # 防止合并分支的自身增量影响 main（只看合并消息）
    track-merge-target: false
    track-merge-message: true  # 启用解析合并提交的消息
    is-main-branch: true
    pre-release-weight: 55000

  feature:  # 对于 feature 分支，不增量版本，只作为开发分支
    regex: ^features?[/-]
    mode: ContinuousDeployment
    increment: None  # 不自动增量
    prevent-increment:
      when-current-commit-tagged: true
      when-branch-merged: false
      of-merged-branch: false

# 配置 SemVer 关键词解析（支持 +semver: 风格，或自定义为 Conventional Commits）
major-version-bump-message: '(?m)^((build|chore|ci|docs|feat|fix|perf|refactor|revert|style|test)(\(.+\))?)(!:.+)|(^.+!:.*)|(\+semver:\s?(breaking|major))'  # major 增量：BREAKING CHANGE 或 +semver: major/breaking
minor-version-bump-message: '(?m)^((feat)(\(.+\))?:).+|\+semver:\s?(feature|minor)'  # minor 增量：feat: 开头 或 +semver: minor/feature
patch-version-bump-message: '(?m)^((build|chore|ci|docs|fix|perf|refactor|revert|style|test)(\(.+\))?:).+|\+semver:\s?(fix|patch)'  # patch 增量：fix: 等 或 +semver: patch/fix
no-bump-message: '\+semver:\s?(none|skip)'  # 跳过增量：+semver: none/skip

ignore:
  sha: []  # 如果需要忽略某些提交 SHA，可以在这里添加
```

### 解释
- **mode: Mainline**：版本计算只基于 main 分支的提交历史。合并提交会被视为 main 上的关键点，不会遍历分支上的所有单个提交。
- **commit-message-incrementing: MergeMessageOnly**：这是核心，确保只检查合并提交的消息（例如 PR 合并时的消息）。如果合并消息包含 SemVer 关键词，才会增量版本；否则保持不变。
- **track-merge-message: true**：在 main 分支上启用解析合并消息。
- **prevent-increment of-merged-branch: true**：防止被合并的分支（如 feature）的提交影响版本，只依赖合并消息。
- **SemVer 关键词配置**：我配置了支持 Conventional Commits（Angular 风格，如 `feat: add new feature` 增 minor，`fix: bug fix` 增 patch，`BREAKING CHANGE:` 或 `feat!: ` 增 major）。同时兼容 +semver: 风格。如果你只想用 +semver:，可以简化 regex 只保留 +semver 部分。
- **其他分支**：feature 分支设置为不增量（increment: None），确保开发时版本稳定，只有合并到 main 后才可能变化。

### 使用建议
1. 将此 GitVersion.yml 放在仓库根目录。
2. 在 PR 合并时，确保合并消息包含关键词，例如：
   - "Merge pull request #123: feat: add new API" → 增 minor。
   - "Merge branch 'fix/bug': +semver: patch" → 增 patch。
   - 如果无关键词或有 +semver: skip → 不增量。
3. 测试：运行 `gitversion /showvariable FullSemVer` 查看计算的版本。
4. 如果你的工作流是 GitHub Flow（feature PR 到 main），这很合适。如果需要调整 regex 或其他，参考 GitVersion 官方文档。

如果实际运行有问题，可以提供更多仓库细节，我再帮调。