# e签宝-基于文件发起签署：多文件 + 多个甲方（我司主体）方案分析

## 1. 需求本质

把当前「一次签署流程 = 多份文件 + 一个我司主体（甲方）自动落章 + 一个商户/机构手动签署」扩展为：

> 一次签署流程 = 多份文件 + 多个我司主体（甲方）各自自动落章 + 一个商户/机构手动签署。

例如：同一批签约里，部分协议由「随行付」落章，部分协议由「天阙」落章。

## 2. e签宝接口天然能力

接口：`POST /v3/sign-flow/create-by-file`

| 能力 | 说明 | 对本次需求的意义 |
|---|---|---|
| `docs` 数组 | 一个流程最多 50 个文件，总大小 ≤500MB | 已支持多文件 |
| `signers` 数组 | 一个流程最多 10 个签署方（可通过「追加签署区」扩展到 50） | 已支持多方签署 |
| 一个 signer 绑定多份文件 | `signers[].signFields[]` 是数组，每个 `signField.fileId` 指向 `docs[]` 中任意文件 | 一个甲方可以盖多份文件 |
| 一个文件被多个签署方签署 | 多个 signer 的 `signFields` 可以指向同一个 `fileId` | 一份协议多章无冲突 |
| 平台方自动落章 | `signerType=1` + `normalSignFieldConfig.autoSign=true` + `assignedSealId=指定印章ID` | 每个甲方只需换 `assignedSealId` |
| 同一 appId 下多枚印章 | e签宝按 appId 归集企业印章，不同主体印章用 `assignedSealId` 区分 | 无需换 appId 即可支持多甲方 |

### 2.1 关键参数传法（多个甲方）

假设一次流程有 2 个甲方：随行付、天阙；1 个乙方：商户/机构。

```json
{
  "docs": [
    { "fileId": "F1", "fileName": "合作服务协议-随行付.pdf", "order": 1 },
    { "fileId": "F2", "fileName": "补充协议-随行付.pdf", "order": 2 },
    { "fileId": "F3", "fileName": "合作服务协议-天阙.pdf", "order": 3 },
    { "fileId": "F4", "fileName": "补充协议-天阙.pdf", "order": 4 }
  ],
  "signers": [
    {
      "signerType": 1,
      "signFields": [
        { "fileId": "F1", "normalSignFieldConfig": { "autoSign": true, "assignedSealId": "随行付印章ID", "signFieldPosition": { "positionPage": "1", "positionX": 300, "positionY": 200 } } },
        { "fileId": "F2", "normalSignFieldConfig": { "autoSign": true, "assignedSealId": "随行付印章ID", "signFieldPosition": { "positionPage": "1", "positionX": 300, "positionY": 200 } } }
      ]
    },
    {
      "signerType": 1,
      "signFields": [
        { "fileId": "F3", "normalSignFieldConfig": { "autoSign": true, "assignedSealId": "天阙印章ID", "signFieldPosition": { "positionPage": "1", "positionX": 300, "positionY": 200 } } },
        { "fileId": "F4", "normalSignFieldConfig": { "autoSign": true, "assignedSealId": "天阙印章ID", "signFieldPosition": { "positionPage": "1", "positionX": 300, "positionY": 200 } } }
      ]
    },
    {
      "signerType": 1,
      "orgSignerInfo": { "orgName": "商户注册名", "orgInfo": { "orgIDCardType": "CRED_ORG_USCC", "orgIDCardNum": "营业执照号" }, "transactorInfo": { "psnAccount": "经办人手机号", "psnInfo": { "psnName": "经办人姓名" } } },
      "signFields": [ ... ]
    }
  ]
}
```

### 2.2 约束与注意事项

1. **appId 仍唯一**：发起方 appId 不变（例如随行付 appId）。多个甲方印章必须都在该 appId 对应的企业空间下；若某主体未在该 appId 下做印章授权或不存在印章，e签宝会报错。
2. **autoSign=true 的平台方签署方**：建议不传 `orgSignerInfo`，只传 `assignedSealId`；e签宝官方说明“不传此字段取默认印章，传入则取指定印章”。若传 `orgName` 与自动落章混用，可能因实名不一致导致报错。
3. **签署区数量**：流程内 `signFields` 总数 ≤300。按 10 文件 × 3 方用印估算，建议封装层限制 ≤30 组，剩余余量用于扩展。
4. **签署方数量**：一次性传入 ≤10，超出需后续调用「追加签署区」；当前需求多为 2-3 个甲方 + 1 乙方，不会触及。
5. **自动落章顺序**：平台方 `autoSign=true` 时，e签宝在流程开启后自动落章，无需签约人访问页面；手动签署方（商户/机构）在所有前置顺序的自动签署完成后才能签署。
6. **通知与重定向**：仍由 `signFlowConfig.notifyUrl`、`redirectConfig.redirectUrl` 统一配置。

## 3. 现状封装层接口的偏差

当前 PRD（`创建签约并获取链接`）在设计上允许传入 10 组签署方，但实际落地逻辑是：

- 只取一个「我司主体」映射到一个甲方 signer；
- 所有文件的所有签章区都挂在这个甲方 signer 下；
- 调用架构 appId 时「有多个先任取 1 个」，说明没有做多 appId/多印章的映射。

因此要实现多甲方，需要修改的不是「能否传多个签署方」，而是：

> **让「我司主体」字段从「全局唯一」变成「按文件/按签章区可配置」**，并把同主体的签章区聚合为同一个 signer，减少 signers 数量。

## 4. 推荐改造方向

保持对外接口结构基本不变，升级内部聚合规则：

1. 业务方在「签约文件」或「签署区信息」中显式指定每个签章区的「我司主体」；
2. 封装层按「我司主体」把签章区分组，生成多个 `signerType=1 + autoSign=true + assignedSealId=对应主体印章ID` 的甲方 signer；
3. 原有乙方（商户/机构）签署方逻辑不变；
4. 校验规则新增：
   - 同一个 `协议外部ID` 下所有签章区必须属于同一个甲方（避免同一份协议同一位置盖两个不同甲方的章，除非业务明确需要）；
   - 甲方数量 ≤ 允许上限；
   - 每个甲方印章 ID 必须存在且已配置。

