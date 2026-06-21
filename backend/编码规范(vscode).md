配置 VS Code 在当前 Python 项目里用 Ruff 做格式化(以及 lint),分两层:**VS Code 扩展层**和**项目配置层**。下面按步骤来。

## 一、确认前置条件

1. VS Code 里已安装 **Ruff** 扩展(发布者是 Astral Software,**不是**老的第三方版本)
2. 项目里可以选择性地装 ruff:
   ```bash
   uv add --dev ruff
   ```
   不装也行,Ruff 扩展自带 ruff 二进制,会自动使用。但**装到项目里更推荐**,可以保证团队/CI 用同一个版本。

## 二、配置 VS Code 使用 Ruff 作为默认格式化工具

在项目根目录创建 `.vscode/settings.json`:

```json
{
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.fixAll.ruff": "explicit",
      "source.organizeImports.ruff": "explicit"
    }
  },
  "ruff.nativeServer": "on"
}
```

这段配置做了三件事:
- **`defaultFormatter`**:Python 文件保存时用 Ruff 格式化(等同 black)
- **`source.fixAll.ruff`**:自动修复 lint 错误(等同 `ruff check --fix`)
- **`source.organizeImports.ruff`**:自动整理 import(等同 isort)

> `"explicit"` 表示只在显式保存(Ctrl+S)时触发,不会在自动保存时触发,体验更可控。
> `ruff.nativeServer: "on"` 启用新的原生 LSP 服务器,比旧的 Python 实现快很多(0.5.3+ 默认就是 on,显式写上更稳)。

如果**不想让项目级配置影响其他人**,也可以放到用户级 `settings.json`(Ctrl+Shift+P → "Preferences: Open User Settings (JSON)"),但项目级更推荐——团队成员 clone 下来就直接生效。

## 三、配置 Ruff 本身的规则(项目级)

Ruff 的规则配置写在 `pyproject.toml` 里。这是 ruff 的"项目宪法",VS Code 扩展会自动读取它。

在 `pyproject.toml` 末尾加上:

```toml
[tool.ruff]
# 行宽,和 black 默认一致
line-length = 100
# 目标 Python 版本(根据你的项目调整)
target-version = "py312"

# 排除目录(默认就排除 .venv 等,这里可以追加)
extend-exclude = [
    "migrations",
    "build",
]

[tool.ruff.lint]
# 启用的规则集
select = [
    "E",    # pycodestyle 错误
    "W",    # pycodestyle 警告
    "F",    # pyflakes
    "I",    # isort(整理 import)
    "B",    # bugbear(常见 bug)
    "C4",   # comprehensions(推导式优化)
    "UP",   # pyupgrade(用新语法)
    "SIM",  # simplify(简化代码)
    "RUF",  # ruff 自己的规则
]
# 忽略的规则
ignore = [
    "E501",  # 行太长(让 formatter 处理)
]

[tool.ruff.lint.per-file-ignores]
# 对特定文件放宽规则
"__init__.py" = ["F401"]     # 允许未使用的 import
"tests/*" = ["S101"]         # 测试里允许 assert

[tool.ruff.format]
# 字符串引号风格:double / single / preserve
quote-style = "double"
# 缩进:space / tab
indent-style = "space"
# 行尾:auto / lf / cr / crlf(WSL 推荐 lf)
line-ending = "lf"
```

**几个常见调整**

- 想保持和 black 完全一致:`line-length = 88`、`quote-style = "double"`
- 想严格一点(推荐新项目):在 `select` 里追加 `"N"` (命名约定)、`"D"` (docstring)、`"ANN"` (类型注解强制)
- 想宽松一点:只保留 `["E", "F", "I"]`

## 四、验证配置生效

**1. 命令行验证**

```bash
uv run ruff check .       # 跑 lint
uv run ruff format .      # 跑格式化(实际改文件)
uv run ruff format --check .  # 只检查,不改
```

**2. VS Code 里验证**

- 随便打开一个 `.py` 文件
- 故意写一段乱代码,比如:
  ```python
  import os,sys
  def foo( x,y ):
      return    x+y
  ```
- 按 **Ctrl+S** 保存,应该立即被格式化成:
  ```python
  import os
  import sys


  def foo(x, y):
      return x + y
  ```
- 如果有 lint 警告,在编辑器左下角/问题面板里能看到红/黄波浪线

**3. 检查 Ruff 用的是哪个版本**

在 VS Code 底部状态栏点击 "Ruff" 文字,或者用命令面板 → "Ruff: Show Server Logs",可以看到它用的 ruff 路径和版本。

## 五、可选:配合 pre-commit(强烈推荐)

让 git commit 前自动跑 ruff,避免脏代码进仓库:

```bash
uv add --dev pre-commit
```

创建 `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.0   # 用 ruff 命令查最新版本号
    hooks:
      - id: ruff           # 跑 lint + 自动修复
        args: [--fix]
      - id: ruff-format    # 跑 format
```

安装 hook:

```bash
uv run pre-commit install
```

之后每次 `git commit` 都会自动跑 ruff,有问题会阻止提交。

## 六、常见坑

**坑 1:保存后没格式化**

检查顺序:
1. 文件右下角语言模式是不是 "Python"
2. `Ctrl+Shift+P` → "Format Document With..." → 看看默认 formatter 是不是 Ruff
3. 看下有没有装别的 formatter(如 Black、autopep8)在抢工作,在 settings 里明确指定 `charliermarsh.ruff`

**坑 2:`source.fixAll` 把 import 顺序改乱了**

把 `organizeImports` 单独配出来,别用 `"source.fixAll": true`(那个会触发所有 source action,可能不止 ruff 一个)。上面我给的配置已经避免了这点。

**坑 3:WSL 2 里扩展找不到 ruff**

确保 Ruff 扩展是装在 **WSL 端**而不是 Windows 端。打开 VS Code 扩展面板,找到 Ruff,如果显示 "Install in WSL: Ubuntu" 按钮,点一下。左下角应该显示 `WSL: Ubuntu`,扩展也应该显示在 "WSL: Ubuntu - Installed" 分类下。

**坑 4:Ruff 版本太老**

旧版本 ruff(< 0.1.0)还没有 `format` 子命令,只能 lint。确认版本:

```bash
uv run ruff --version
```

低于 0.5.0 建议升级:

```bash
uv add --dev "ruff>=0.8"
```

## 一键配置脚本

如果想偷懒,在项目根目录跑一遍这个就齐活了:

```bash
# 1. 装依赖
uv add --dev ruff pre-commit

# 2. 创建 VS Code 配置
mkdir -p .vscode
cat > .vscode/settings.json << 'EOF'
{
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.fixAll.ruff": "explicit",
      "source.organizeImports.ruff": "explicit"
    }
  },
  "ruff.nativeServer": "on"
}
EOF

# 3. 追加 ruff 配置到 pyproject.toml
cat >> pyproject.toml << 'EOF'

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "W", "F", "I", "B", "C4", "UP", "SIM", "RUF"]
ignore = ["E501"]

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
line-ending = "lf"
EOF

# 4. 先全量格式化一次
uv run ruff check --fix .
uv run ruff format .
```

跑完之后重启一下 VS Code,Ctrl+S 就能享受 Ruff 的速度了。

有任何报错或者想调整规则严格度,告诉我具体需求,我帮你调。