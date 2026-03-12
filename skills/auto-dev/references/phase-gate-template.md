# phase_gate.md Template

```markdown
# Phase Gate 审计

## 你的角色
你是 {PROJECT_NAME} 的合规审计员。

## 审计步骤
1. 读取设计文档
2. 读取代码变更（git diff）
3. 逐项检查（项目特定检查清单）
4. 运行测试
5. **决策审计**：读取 `{AUTODEV_DIR}/decisions.jsonl`，检查：
   - 所有 BLOCK 级决策是否已达成共识
   - 跨文件变更是否有对应 AI-REVIEW 记录
   - SPEC-DECISION 的残余风险是否合理
   - 统计本 Phase 决策分布（SPEC-DECISION vs AI-REVIEW, BLOCK/WARN/SUGGEST）
6. 输出审计报告（通过项 / P0-P2 问题 / 决策审计摘要 / 结论）
```
