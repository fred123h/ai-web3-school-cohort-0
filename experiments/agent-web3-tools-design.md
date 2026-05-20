# Agent Web3 工具设计规范

> 核心原则：**读写分离、草稿不发送、权限必校验、操作可审计**

---

## 📐 设计原则

| 层级 | 职责 | 关键特征 |
|------|------|---------|
| **Read** | 读取链上状态 | 无状态、无签名、无风险 |
| **Draft** | 生成交易数据 | 不发送、可预览、可撤销 |
| **Sign** | 签名授权 | 用户确认、私钥不暴露 |
| **Submit** | 发送上链 | 最终执行、不可逆 |
| **Log** | 记录审计 | 全程追踪、可回溯 |

---

## 🛠 Tool 1: getEthBalance（只读工具）

### 功能
读取某地址在某链上的原生代币（ETH/MATIC/BNB 等）余额。

### 输入 Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["chainId", "address"],
  "properties": {
    "chainId": {
      "type": "integer",
      "description": "Chain ID (e.g., 1 for Ethereum, 137 for Polygon)",
      "minimum": 1,
      "examples": [1, 137, 56, 42161]
    },
    "address": {
      "type": "string",
      "pattern": "^0x[a-fA-F0-9]{40}$",
      "description": "Address to query"
    },
    "blockTag": {
      "type": "string",
      "enum": ["latest", "earliest", "pending"],
      "default": "latest",
      "description": "Block to query at"
    }
  },
  "additionalProperties": false
}
```

### 输出字段

```json
{
  "type": "object",
  "properties": {
    "success": { "type": "boolean" },
    "chainId": { "type": "integer" },
    "address": { "type": "string" },
    "blockNumber": { 
      "type": "integer",
      "description": "Block number at which balance was queried"
    },
    "balance": {
      "type": "object",
      "properties": {
        "wei": { "type": "string", "description": "Raw balance in wei" },
        "ether": { "type": "string", "description": "Human-readable balance" },
        "decimals": { "type": "integer", "default": 18 }
      }
    },
    "nativeToken": {
      "type": "object",
      "properties": {
        "symbol": { "type": "string" },
        "name": { "type": "string" }
      }
    },
    "queryTime": { "type": "string", "format": "date-time" }
  }
}
```

### 输出示例

```json
{
  "success": true,
  "chainId": 1,
  "address": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
  "blockNumber": 25060241,
  "balance": {
    "wei": "1234567890123456789",
    "ether": "1.234567890123456789",
    "decimals": 18
  },
  "nativeToken": {
    "symbol": "ETH",
    "name": "Ether"
  },
  "queryTime": "2026-05-19T16:30:00.000Z"
}
```

### 错误类型

| 错误码 | 描述 | 触发条件 |
|--------|------|---------|
| `INVALID_ADDRESS` | 地址格式无效 | 地址不符合 0x + 40 hex 格式 |
| `UNSUPPORTED_CHAIN` | 链不支持 | chainId 未在支持的链列表中 |
| `RPC_ERROR` | RPC 节点错误 | 节点超时、无响应、返回错误 |
| `RATE_LIMITED` | 请求频率限制 | 触发 RPC 提供商限流 |
| `BLOCK_NOT_FOUND` | 区块不存在 | blockTag 指向的区块不存在（极端情况） |

### 日志字段

```json
{
  "tool": "getEthBalance",
  "operation": "READ",
  "chainId": 1,
  "address": "0xd8dA...",
  "blockTag": "latest",
  "blockNumber": 25060241,
  "balanceWei": "1234567890123456789",
  "rpcEndpoint": "https://ethereum.publicnode.com",
  "durationMs": 125,
  "status": "SUCCESS",
  "timestamp": "2026-05-19T16:30:00.000Z"
}
```

---

## 🛠 Tool 2: getErc20Allowance（合约读取工具）

### 功能
读取 ERC-20 合约中 `owner` 授权给 `spender` 的额度。

### 输入 Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["chainId", "tokenAddress", "owner", "spender"],
  "properties": {
    "chainId": {
      "type": "integer",
      "description": "Chain ID",
      "minimum": 1
    },
    "tokenAddress": {
      "type": "string",
      "pattern": "^0x[a-fA-F0-9]{40}$",
      "description": "ERC-20 token contract address"
    },
    "owner": {
      "type": "string",
      "pattern": "^0x[a-fA-F0-9]{40}$",
      "description": "Address that owns the tokens"
    },
    "spender": {
      "type": "string",
      "pattern": "^0x[a-fA-F0-9]{40}$",
      "description": "Address that is approved to spend"
    },
    "blockTag": {
      "type": "string",
      "enum": ["latest", "earliest", "pending"],
      "default": "latest"
    }
  },
  "additionalProperties": false
}
```

### 输出字段

```json
{
  "type": "object",
  "properties": {
    "success": { "type": "boolean" },
    "chainId": { "type": "integer" },
    "tokenAddress": { "type": "string" },
    "token": {
      "type": "object",
      "properties": {
        "symbol": { "type": "string" },
        "name": { "type": "string" },
        "decimals": { "type": "integer" }
      },
      "description": "Token metadata (may be cached)"
    },
    "owner": { "type": "string" },
    "spender": { 
      "type": "string",
      "description": "Spender address with optional label if known"
    },
    "allowance": {
      "type": "object",
      "properties": {
        "raw": { "type": "string", "description": "Raw allowance in token units" },
        "human": { "type": "string", "description": "Human-readable allowance" },
        "isUnlimited": { 
          "type": "boolean",
          "description": "True if allowance is MaxUint256"
        }
      }
    },
    "blockNumber": { "type": "integer" },
    "queryTime": { "type": "string", "format": "date-time" }
  }
}
```

### 输出示例

```json
{
  "success": true,
  "chainId": 1,
  "tokenAddress": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
  "token": {
    "symbol": "USDC",
    "name": "USD Coin",
    "decimals": 6
  },
  "owner": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
  "spender": "0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45",
  "allowance": {
    "raw": "115792089237316195423570985008687907853269984665640564039457584007913129639935",
    "human": "Unlimited",
    "isUnlimited": true
  },
  "blockNumber": 25060241,
  "queryTime": "2026-05-19T16:30:00.000Z"
}
```

### 错误类型

| 错误码 | 描述 | 触发条件 |
|--------|------|---------|
| `INVALID_ADDRESS` | 地址格式无效 | 任一地址格式错误 |
| `UNSUPPORTED_CHAIN` | 链不支持 | chainId 未配置 |
| `NOT_ERC20_CONTRACT` | 非 ERC-20 合约 | 合约没有 `allowance` 方法 |
| `CONTRACT_CALL_FAILED` | 合约调用失败 | 合约执行 revert |
| `RPC_ERROR` | RPC 节点错误 | 节点问题 |
| `TOKEN_METADATA_MISSING` | 代币元数据缺失 | 无法获取 symbol/decimals（非致命，允许继续） |

### 日志字段

```json
{
  "tool": "getErc20Allowance",
  "operation": "READ",
  "chainId": 1,
  "tokenAddress": "0xa0b86...",
  "tokenSymbol": "USDC",
  "owner": "0xd8dA...",
  "spender": "0x68b3...",
  "allowanceRaw": "115792...39935",
  "isUnlimited": true,
  "rpcEndpoint": "https://ethereum.publicnode.com",
  "durationMs": 187,
  "status": "SUCCESS",
  "timestamp": "2026-05-19T16:30:00.000Z"
}
```

---

## 🛠 Tool 3: buildErc20ApproveCalldata（交易草稿工具）

### 功能
生成 ERC-20 `approve(spender, amount)` 的 calldata，**不发送交易**。

### 输入 Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["chainId", "tokenAddress", "spender", "amount"],
  "properties": {
    "chainId": {
      "type": "integer",
      "description": "Chain ID",
      "minimum": 1
    },
    "tokenAddress": {
      "type": "string",
      "pattern": "^0x[a-fA-F0-9]{40}$",
      "description": "ERC-20 token contract address"
    },
    "spender": {
      "type": "string",
      "pattern": "^0x[a-fA-F0-9]{40}$",
      "description": "Address to approve"
    },
    "amount": {
      "oneOf": [
        { "type": "string", "pattern": "^\\d+$" },
        { "type": "string", "enum": ["MAX", "ZERO"] }
      ],
      "description": "Amount to approve (raw units), or 'MAX' for unlimited, or 'ZERO' to revoke"
    },
    "dryRun": {
      "type": "boolean",
      "default": true,
      "description": "If true, simulate the transaction without broadcasting"
    }
  },
  "additionalProperties": false
}
```

### 输出字段

```json
{
  "type": "object",
  "properties": {
    "success": { "type": "boolean" },
    "operation": { 
      "type": "string", 
      "const": "DRAFT",
      "description": "Always DRAFT - transaction not sent"
    },
    "chainId": { "type": "integer" },
    "tokenAddress": { "type": "string" },
    "token": {
      "type": "object",
      "properties": {
        "symbol": { "type": "string" },
        "decimals": { "type": "integer" }
      }
    },
    "spender": { "type": "string" },
    "amount": {
      "type": "object",
      "properties": {
        "raw": { "type": "string" },
        "human": { "type": "string" },
        "type": { 
          "type": "string",
          "enum": ["EXACT", "UNLIMITED", "REVOKE"]
        }
      }
    },
    "txRequest": {
      "type": "object",
      "description": "Unsigned transaction request",
      "properties": {
        "to": { "type": "string" },
        "from": { "type": "string", "description": "Expected sender (not yet signed)" },
        "data": { "type": "string", "description": "Encoded calldata" },
        "value": { "type": "string", "const": "0x0" },
        "gasLimit": { "type": "string", "description": "Estimated gas (optional)" },
        "gasPrice": { "type": "string", "description": "Current gas price (optional)" }
      }
    },
    "decodedCalldata": {
      "type": "object",
      "description": "Human-readable calldata breakdown",
      "properties": {
        "method": { "type": "string", "const": "approve" },
        "abi": { "type": "string", "const": "approve(address,uint256)" },
        "params": {
          "type": "object",
          "properties": {
            "spender": { "type": "string" },
            "amount": { "type": "string" }
          }
        }
      }
    },
    "simulationResult": {
      "type": "object",
      "description": "Optional dry-run simulation result",
      "properties": {
        "success": { "type": "boolean" },
        "gasUsed": { "type": "integer" },
        "logs": { "type": "array" },
        "error": { "type": "string" }
      }
    },
    "nextStep": {
      "type": "string",
      "description": "What action is needed next",
      "enum": ["SIGN_AND_SUBMIT", "REVIEW_AND_CONFIRM", "REJECTED_BY_POLICY", "NEEDS_APPROVAL"]
    },
    "draftId": {
      "type": "string",
      "description": "Unique ID for this draft, can be referenced later"
    },
    "createdAt": { "type": "string", "format": "date-time" }
  }
}
```

### 输出示例

```json
{
  "success": true,
  "operation": "DRAFT",
  "chainId": 1,
  "tokenAddress": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
  "token": {
    "symbol": "USDC",
    "decimals": 6
  },
  "spender": "0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45",
  "amount": {
    "raw": "1000000000",
    "human": "1000.0",
    "type": "EXACT"
  },
  "txRequest": {
    "to": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
    "from": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
    "data": "0x095ea7b200000000000000000000000068b3465833fb72a70ecdf485e0e4c7bd8665fc45000000000000000000000000000000000000000000000000000000003b9aca00",
    "value": "0x0",
    "gasLimit": "0xfde8",
    "gasPrice": "0x5d21dba00"
  },
  "decodedCalldata": {
    "method": "approve",
    "abi": "approve(address,uint256)",
    "params": {
      "spender": "0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45",
      "amount": "1000000000"
    }
  },
  "simulationResult": {
    "success": true,
    "gasUsed": 46271,
    "logs": [
      {
        "address": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
        "topics": ["0x8c5be1e5ebec7d5bd14f71427d1e84f3dd0314c0f7b2291e5b200ac8c7c3b925", "..."],
        "data": "0x..."
      }
    ]
  },
  "nextStep": "SIGN_AND_SUBMIT",
  "draftId": "draft_20260519_163000_a1b2c3d4",
  "createdAt": "2026-05-19T16:30:00.000Z"
}
```

### 错误类型

| 错误码 | 描述 | 触发条件 |
|--------|------|---------|
| `INVALID_ADDRESS` | 地址格式无效 | token 或 spender 地址格式错误 |
| `INVALID_AMOUNT` | 金额无效 | 金额为负数或格式不正确 |
| `NOT_ERC20_CONTRACT` | 非 ERC-20 合约 | 合约没有 `approve` 方法 |
| `SIMULATION_FAILED` | 模拟失败 | dry-run 时交易 revert |
| `UNSUPPORTED_CHAIN` | 链不支持 | chainId 未配置 |
| `ABI_ENCODING_ERROR` | ABI 编码错误 | calldata 编码失败 |

### 日志字段

```json
{
  "tool": "buildErc20ApproveCalldata",
  "operation": "DRAFT",
  "chainId": 1,
  "tokenAddress": "0xa0b86...",
  "tokenSymbol": "USDC",
  "spender": "0x68b3...",
  "amountRaw": "1000000000",
  "amountType": "EXACT",
  "dryRun": true,
  "simulationSuccess": true,
  "gasLimitEstimate": 46271,
  "draftId": "draft_20260519_163000_a1b2c3d4",
  "durationMs": 342,
  "status": "SUCCESS",
  "timestamp": "2026-05-19T16:30:00.000Z"
}
```

---

## 🛠 Tool 4: submitErc20Approve（写交易工具 + 权限规则）

### 功能
发送 ERC-20 approve 交易上链，**受权限规则约束**。

### 权限规则

```yaml
permission_rules:
  # 允许的 token 列表
  allowed_tokens:
    - chainId: 1
      address: "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48"
      symbol: USDC
      max_amount: null  # null = unlimited
    - chainId: 1
      address: "0xdac17f958d2ee523a2206206994597c13d831ec7"
      symbol: USDT
      max_amount: null
    - chainId: 137
      address: "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174"
      symbol: USDC
      max_amount: "10000000000"  # 10,000 USDC max
  
  # 允许的 spender 列表
  allowed_spenders:
    - chainId: 1
      address: "0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45"
      label: "Uniswap V3 Router"
      require_confirmation: false  # 自动批准
    - chainId: 1
      address: "0x1111111254EEB25477B68fb85Ed929f73A960582"
      label: "1inch Router"
      require_confirmation: false
    - chainId: 1
      address: "0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D"
      label: "Uniswap V2 Router"
      require_confirmation: true   # 需要用户确认
    
  # 默认行为
  default_policy:
    # 未在允许列表中的 token
    unknown_token: REJECT
    # 未在允许列表中的 spender
    unknown_spender: REJECT
    # 超过 max_amount
    over_max_amount: CONFIRM  # 需要用户确认
    # unlimited approve (MAX)
    unlimited_approve: CONFIRM  # 需要用户确认
    
  # 全局限制
  global_limits:
    max_pending_approvals: 3
    require_confirmation_for_new_spenders: true
```

### 输入 Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["chainId", "tokenAddress", "spender", "amount", "from"],
  "properties": {
    "chainId": {
      "type": "integer",
      "minimum": 1
    },
    "tokenAddress": {
      "type": "string",
      "pattern": "^0x[a-fA-F0-9]{40}$"
    },
    "spender": {
      "type": "string",
      "pattern": "^0x[a-fA-F0-9]{40}$"
    },
    "amount": {
      "oneOf": [
        { "type": "string", "pattern": "^\\d+$" },
        { "type": "string", "enum": ["MAX", "ZERO"] }
      ]
    },
    "from": {
      "type": "string",
      "pattern": "^0x[a-fA-F0-9]{40}$",
      "description": "Sender address (must match signing key)"
    },
    "draftId": {
      "type": "string",
      "description": "Reference to a previously created draft"
    },
    "confirmationToken": {
      "type": "string",
      "description": "Token from user confirmation step (if required)"
    },
    "gasSettings": {
      "type": "object",
      "properties": {
        "maxFeePerGas": { "type": "string" },
        "maxPriorityFeePerGas": { "type": "string" },
        "gasLimit": { "type": "string" }
      }
    }
  },
  "additionalProperties": false
}
```

### 输出字段

```json
{
  "type": "object",
  "properties": {
    "success": { "type": "boolean" },
    "operation": { 
      "type": "string",
      "enum": ["SUBMITTED", "PENDING_CONFIRMATION", "REJECTED"]
    },
    "permissionCheckResult": {
      "type": "object",
      "properties": {
        "passed": { "type": "boolean" },
        "tokenAllowed": { "type": "boolean" },
        "spenderAllowed": { "type": "boolean" },
        "amountWithinLimit": { "type": "boolean" },
        "requiresConfirmation": { "type": "boolean" },
        "rejectionReason": { "type": "string" }
      }
    },
    "chainId": { "type": "integer" },
    "tokenAddress": { "type": "string" },
    "token": { "type": "object" },
    "spender": { "type": "string" },
    "amount": { "type": "object" },
    "txRequest": { "type": "object" },
    "txHash": { 
      "type": "string",
      "description": "Transaction hash if submitted"
    },
    "txStatus": {
      "type": "string",
      "enum": ["PENDING", "SUBMITTED", "CONFIRMED", "FAILED"],
      "description": "Transaction status"
    },
    "blockNumber": {
      "type": "integer",
      "description": "Block number if confirmed"
    },
    "confirmationPending": {
      "type": "object",
      "description": "Present if waiting for user confirmation",
      "properties": {
        "confirmationUrl": { "type": "string" },
        "expiresAt": { "type": "string", "format": "date-time" },
        "pendingId": { "type": "string" }
      }
    },
    "explorerUrl": { "type": "string" },
    "submittedAt": { "type": "string", "format": "date-time" },
    "confirmedAt": { "type": "string", "format": "date-time" }
  }
}
```

### 输出示例

**成功提交**:
```json
{
  "success": true,
  "operation": "SUBMITTED",
  "permissionCheckResult": {
    "passed": true,
    "tokenAllowed": true,
    "spenderAllowed": true,
    "amountWithinLimit": true,
    "requiresConfirmation": false
  },
  "chainId": 1,
  "tokenAddress": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
  "token": { "symbol": "USDC", "decimals": 6 },
  "spender": "0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45",
  "amount": {
    "raw": "1000000000",
    "human": "1000.0",
    "type": "EXACT"
  },
  "txRequest": {
    "to": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
    "from": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
    "data": "0x095ea7b2...",
    "value": "0x0"
  },
  "txHash": "0xabc123...",
  "txStatus": "SUBMITTED",
  "explorerUrl": "https://etherscan.io/tx/0xabc123...",
  "submittedAt": "2026-05-19T16:30:00.000Z"
}
```

**等待确认**:
```json
{
  "success": false,
  "operation": "PENDING_CONFIRMATION",
  "permissionCheckResult": {
    "passed": false,
    "tokenAllowed": true,
    "spenderAllowed": true,
    "amountWithinLimit": true,
    "requiresConfirmation": true,
    "rejectionReason": "Unlimited approve requires user confirmation"
  },
  "chainId": 1,
  "tokenAddress": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
  "spender": "0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D",
  "amount": { "type": "UNLIMITED" },
  "confirmationPending": {
    "confirmationUrl": "https://agent.example/confirm/pending_xyz",
    "expiresAt": "2026-05-19T16:45:00.000Z",
    "pendingId": "pending_xyz"
  }
}
```

**被拒绝**:
```json
{
  "success": false,
  "operation": "REJECTED",
  "permissionCheckResult": {
    "passed": false,
    "tokenAllowed": false,
    "spenderAllowed": false,
    "amountWithinLimit": false,
    "requiresConfirmation": false,
    "rejectionReason": "Token 0x123... not in allowed list and spender 0x456... not in allowed list"
  }
}
```

### 错误类型

| 错误码 | 描述 | 触发条件 |
|--------|------|---------|
| `UNSUPPORTED_CHAIN` | 链不支持 | chainId 未配置 |
| `TOKEN_NOT_ALLOWED` | Token 不在允许列表 | token 不在 allowed_tokens |
| `SPENDER_NOT_ALLOWED` | Spender 不在允许列表 | spender 不在 allowed_spenders |
| `AMOUNT_EXCEEDS_LIMIT` | 金额超限 | 超过 max_amount |
| `UNLIMITED_APPROVE_REQUIRES_CONFIRMATION` | 无限授权需确认 | amount=MAX 且规则要求确认 |
| `NEEDS_CONFIRMATION` | 需要用户确认 | 规则要求确认但未提供 confirmationToken |
| `CONFIRMATION_EXPIRED` | 确认已过期 | confirmationToken 过期 |
| `INVALID_CONFIRMATION_TOKEN` | 确认令牌无效 | token 无效或已使用 |
| `MAX_PENDING_EXCEEDED` | 待确认数超限 | pending approvals 超过限制 |
| `SIGNING_FAILED` | 签名失败 | 私钥问题或用户取消 |
| `TX_SUBMIT_FAILED` | 交易提交失败 | RPC 错误、nonce 问题等 |
| `TX_REVERTED` | 交易回滚 | 交易上链后执行失败 |

### 日志字段

```json
{
  "tool": "submitErc20Approve",
  "operation": "SUBMIT",
  "chainId": 1,
  "tokenAddress": "0xa0b86...",
  "tokenSymbol": "USDC",
  "spender": "0x68b3...",
  "spenderLabel": "Uniswap V3 Router",
  "from": "0xd8dA...",
  "amountRaw": "1000000000",
  "amountType": "EXACT",
  "permissionCheck": {
    "passed": true,
    "tokenAllowed": true,
    "spenderAllowed": true,
    "amountWithinLimit": true,
    "requiresConfirmation": false
  },
  "txHash": "0xabc123...",
  "gasUsed": 46271,
  "gasPrice": "25 Gwei",
  "txFeeWei": "1156775000000000",
  "txStatus": "SUBMITTED",
  "durationMs": 1523,
  "status": "SUCCESS",
  "timestamp": "2026-05-19T16:30:00.000Z"
}
```

---

## 📋 工具层级关系

```
                    ┌─────────────────────────────────────────────────┐
                    │                  AGENT LAYER                    │
                    │  (Reasoning, Intent, User Communication)        │
                    └─────────────────────┬───────────────────────────┘
                                          │
                    ┌─────────────────────▼───────────────────────────┐
                    │               TOOL ORCHESTRATION                 │
                    │  (Sequencing, State Management, Rollback)        │
                    └─────────────────────┬───────────────────────────┘
                                          │
        ┌─────────────────────────────────┼─────────────────────────────────┐
        │                                 │                                 │
        ▼                                 ▼                                 ▼
┌───────────────┐               ┌───────────────┐               ┌───────────────┐
│   READ LAYER  │               │   DRAFT LAYER │               │  WRITE LAYER  │
│               │               │               │               │               │
│ getEthBalance │               │ buildApprove  │               │ submitApprove │
│ getAllowance  │               │   Calldata    │               │               │
│ getTokenInfo  │               │               │               │               │
│               │               │               │               │               │
│ No signing    │               │ No sending    │               │ Requires:     │
│ No risk       │               │ Can preview   │               │ - Permission  │
│ Instant       │               │ Can simulate  │               │ - Signature   │
└───────────────┘               └───────────────┘               │ - Confirmation│
                                                                └───────────────┘
        │                                 │                                 │
        └─────────────────────────────────┼─────────────────────────────────┘
                                          │
                    ┌─────────────────────▼───────────────────────────┐
                    │                   LOG LAYER                      │
                    │  (Audit Trail, Analytics, Debug, Compliance)     │
                    └─────────────────────────────────────────────────┘
```

---

## 🔄 完整工作流示例

### 场景：用户要求授权 1000 USDC 给 Uniswap V3 Router

```
1. [READ] getEthBalance
   └─ 检查用户 ETH 余额（是否有足够 gas）

2. [READ] getErc20Allowance
   └─ 检查当前授权额度
   └─ 如果已足够，提示用户无需操作

3. [DRAFT] buildErc20ApproveCalldata
   └─ 生成 approve calldata
   └─ 执行 dry-run 模拟
   └─ 返回 draftId

4. [WRITE - Phase 1] submitErc20Approve
   └─ 权限检查
      ├─ USDC 在允许列表 ✓
      ├─ Uniswap V3 Router 在允许列表 ✓
      └─ 自动批准（require_confirmation: false）✓
   └─ 无需用户确认
   └─ 签名（调用 wallet provider）
   └─ 提交交易

5. [LOG] 记录完整操作日志
   └─ 包括权限检查结果、签名、交易哈希
```

### 场景：用户要求无限授权给 Uniswap V2 Router

```
1. [READ] getErc20Allowance
   └─ 当前授权额度检查

2. [DRAFT] buildErc20ApproveCalldata
   └─ amount = "MAX"
   └─ 生成 unlimited approve calldata
   └─ 返回 draftId

3. [WRITE - Phase 1] submitErc20Approve
   └─ 权限检查
      ├─ USDC 在允许列表 ✓
      ├─ Uniswap V2 Router 在允许列表 ✓
      └─ require_confirmation: true ⚠️
   └─ 返回 pending 状态
   └─ 生成确认链接

4. [USER CONFIRMATION]
   └─ 用户在确认页面查看详情
   └─ 用户点击"确认"
   └─ 生成 confirmationToken

5. [WRITE - Phase 2] submitErc20Approve (带 confirmationToken)
   └─ 验证 confirmationToken
   └─ 签名
   └─ 提交交易

6. [LOG] 记录确认流程和最终结果
```

---

## 🎯 设计要点总结

### 读写分离
- **Read tools**: 无副作用、无签名、可缓存
- **Draft tools**: 生成数据、不发送、可预览
- **Write tools**: 有副作用、需权限、需签名

### 权限控制
- **白名单机制**: token、spender 都需在允许列表
- **分级授权**: 自动批准 vs 用户确认
- **额度限制**: max_amount 控制
- **特殊规则**: unlimited approve 需额外确认

### 审计追踪
- **全程日志**: 每一步操作都记录
- **上下文关联**: draftId 关联草稿和提交
- **权限审计**: 记录权限检查结果
- **交易追踪**: 从提交到确认的完整生命周期

### 错误处理
- **分类错误**: 链上错误、权限错误、系统错误分开
- **用户友好**: 错误信息可读，包含建议
- **可恢复**: 明确哪些错误可重试

---

*Version: 1.0*
*Last updated: 2026-05-19*
