---
title: 用 Claude Code 做自动化代码迁移 Agent
author: "@peterfei"
date: 2026-06
stack: Claude Code, AST, jscodeshift, TypeScript, Git
difficulty: 专家
---

## 场景

每个技术团队都会经历框架升级的痛苦。以 Vue 2 → Vue 3 迁移为例：

1. Options API → Composition API，写法完全不同
2. `Vue.prototype.$xxx` → `app.config.globalProperties`，全局 API 全改
3. `this.$emit`、`this.$slots`、`this.$listeners` 全部变化
4. `vuex` → `pinia`，状态管理方案都要换
5. 第三方组件库（Element UI → Element Plus）API 不兼容

纯手工迁移：一个中型项目（200+ 组件）需要 2-3 个开发干 2 个月。
而且容易漏改、改错，上线后出兼容性问题。

目标是：**让 Agent 自动分析代码差异，执行批量迁移，并验证迁移后的正确性**。

---

## 实现

### 架构

```
输入：Vue 2 项目源码
                    ┌─────────────────────┐
                    │  代码分析阶段          │
  Claude Code ─────→│  ├─ 识别 Vue 2 语法   │
                    │  ├─ 扫描淘汰 API 使用  │
                    │  └─ 生成迁移清单       │
                    └─────────┬───────────┘
                              ↓
                    ┌─────────────────────┐
                    │  批量执行阶段          │
                    │  ├─ AST 转换 (jscodeshift)│
                    │  ├─ 正则替换 (确定性)   │
                    │  └─ AI 补全 (复杂模式)  │
                    └─────────┬───────────┘
                              ↓
                    ┌─────────────────────┐
                    │  验证阶段             │
                    │  ├─ 编译检查           │
                    │  ├─ 单元测试运行        │
                    │  └─ diff review        │
                    └─────────────────────┘

输出：迁移后的 Vue 3 项目 + 迁移报告
```

### 核心脚本

```bash
#!/bin/bash
# migrate.sh — Vue 2 → Vue 3 自动迁移

PROJECT=$1
OUTPUT="${PROJECT}-v3"

cp -r "$PROJECT" "$OUTPUT"
cd "$OUTPUT"

# Step 1: 使用 jscodeshift 做确定性转换
npx vue-codemod --force .

# Step 2: AI 驱动的复杂模式迁移
claude code --print --prompt "
你是一个 Vue 迁移专家。正在将 Vue 2 项目迁移到 Vue 3。

## 需处理的模式

1. **Options API → Composition API**
   \`\`\`diff
   - export default { data() { return { count: 0 } }, methods: { add() { this.count++ } } }
   + import { ref } from 'vue'
   + export default { setup() { const count = ref(0); const add = () => count.value++; return { count, add } } }
   \`\`\`

2. **this.$emit → emit 参数**
   \`\`\`diff
   - this.\$emit('change', val)
   + emit('change', val)  // setup 的第二个参数
   \`\`\`

3. **Vue.prototype → app.config.globalProperties**
   \`\`\`diff
   - Vue.prototype.\$api = api
   + app.config.globalProperties.\$api = api
   \`\`\`

4. **Vuex → Pinia**
   \`\`\`diff
   - this.\$store.commit('setUser', user)
   + import { useUserStore } from '@/stores/user'
   + const userStore = useUserStore()
   + userStore.setUser(user)
   \`\`\`

## 项目文件列表
$(find src -name '*.vue' -o -name '*.ts' -o -name '*.js' | head -50)

请逐文件分析并输出迁移后的完整代码。先处理 utils 和 stores，再处理组件。
" > /tmp/migration-report.md

# Step 3: 验证
npm run build 2>&1 | tee /tmp/build.log
BUILD_STATUS=$?

if [ $BUILD_STATUS -ne 0 ]; then
  echo "编译失败，正在修复..."

  claude code --print --prompt "
  编译失败，错误信息如下：
  $(cat /tmp/build.log | tail -50)

  请分析错误原因并给出修复方案。只修复编译错误，不要改其他逻辑。
  "
fi

# Step 4: 运行测试
npm run test 2>&1 | tee /tmp/test.log

echo "=== 迁移完成 ==="
echo "项目: $OUTPUT"
echo "编译: $([ $BUILD_STATUS -eq 0 ] && echo '通过' || echo '失败')"
echo "测试: $(grep -c 'PASS\|✓' /tmp/test.log || echo '待确认')"
```

### 增量迁移策略

对于大型项目，全量迁移风险太高。建议分模块进行：

```bash
# 按目录分批迁移
MODULES=("src/components/common" "src/views/dashboard" "src/views/settings")

for MOD in "${MODULES[@]}"; do
  echo "正在迁移: $MOD"

  claude code --print --prompt "
  请分析以下模块，评估迁移到 Vue 3 的工作量：

  模块路径：$MOD
  文件数：$(find $MOD -name '*.vue' | wc -l)

  输出格式：
  - 文件数
  - 预计迁移工作量（人天）
  - 高风险文件（需人工重点审查）
  - 建议的迁移顺序
  " > "/tmp/migration-plan-$(basename $MOD).md"

  # 人工确认后再执行
  echo "请确认迁移 $MOD？(y/n)"
  read CONFIRM
  if [ "$CONFIRM" = "y" ]; then
    claude code --prompt "将 $MOD 下的所有 Vue 2 组件迁移到 Vue 3"
  fi
done
```

---

## 效果

在一个 180+ 组件的生产项目上验证（Vue 2.7 → Vue 3.4 + Element Plus）：

| 指标 | 数据 |
|------|------|
| 自动迁移率 | 73%（无需人工干预） |
| 人工修复率 | 22%（轻微手动调整） |
| 需重写 | 5%（复杂逻辑需要人工重写） |
| 编译一次性通过 | 首次 61%，自修复后 89% |
| 测试通过率 | 迁移后 94%（与迁移前持平） |
| 总耗时 | AI 分析 2h + 人工审查 2 天 |
| 对比纯人工 | **节省约 80% 工作量** |

**迁移前后对比**：

| 指标 | Vue 2 人工迁移（估算） | AI Agent 迁移 |
|------|----------------------|---------------|
| 工期 | 2 人 × 2 个月 | 2 人 × 1 周 |
| 线上 bug | 3-5 个（历史经验） | 1 个（mixin 嵌套未处理） |
| 代码一致性 | 依赖个人水平 | 统一风格 |
| 迁移文档 | 无 | 自动化生成变更记录 |

---

## 要点

1. **确定性规则 + AI 模糊匹配** — 纯正则/jscodeshift 能处理 40%（API 重命名、import 路径），但 Options → Composition API 的转换需要 AI 理解上下文。先用工具做确定性转换，再用 AI 补复杂场景。

2. **先测试后迁移** — 迁移前确保项目有完整的单元测试和 E2E 测试。没有测试的项目，先让 Agent 生成测试再迁移。迁移后的正确性靠测试验证，不是靠人眼 review。

3. **分批策略** — 永远不要一次性迁移整个项目。按模块分批：utils/工具函数 → stores → 通用组件 → 页面组件。每批迁移后都跑测试 + 构建。

4. **坑：mixin 展开** — Vue 2 的 mixin 在 Composition API 中没有直接对应。`this.mixinMethod()` 需要改成 `import { useMixin } from './mixin'; const { mixinMethod } = useMixin();`。这块是目前 Agent 处理最差的部分，需要人工重点 review。

5. **diff review 不能省** — Agent 迁移完生成 git diff，开发者 review diff 比 review 最终代码更有效。只看变更的部分，确认没有引入不该改的修改。

---

## 扩展思路

- Python 2 → Python 3 迁移
- jQuery → React/Vue 重构
- JavaScript → TypeScript 迁移（类型推断 + 自动加类型注解）
- Webpack → Vite 构建迁移（配置转换 + 插件替换）
- REST API → GraphQL 迁移

---

## 相关资源

- [vue-codemod — Vue 官方迁移工具](https://github.com/vuejs/vue-codemod)
- [jscodeshift — AST 转换工具包](https://github.com/facebook/jscodeshift)
- [Vue 2 → Vue 3 迁移指南](https://v3-migration.vuejs.org)
- [Element Plus（Element UI 的 Vue 3 版本）](https://element-plus.org)
