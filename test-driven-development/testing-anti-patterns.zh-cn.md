# 测试反模式

**何时加载此参考文档：** 编写或修改测试、添加mock、或想要在生产代码中添加仅用于测试的方法时。

## 概述

测试必须验证真实行为，而不是mock行为。Mock是隔离手段，不是被测试的对象。

**核心原则：** 测试代码做了什么，而不是mock做了什么。

**严格遵循TDD可以避免这些反模式：**
- 先写测试 → 明确测试目标，避免测试mock
- 看它失败 → 验证测试的是真实行为
- 最小实现 → 不会添加仅用于测试的代码
- 真实优先 → 理解依赖后再考虑mock

## 铁律

```
1. 禁止测试mock行为
2. 禁止在生产类中添加仅用于测试的方法
3. 禁止在不理解依赖的情况下使用mock
```

## 反模式1：测试Mock行为

**违规示例：**
```typescript
// ❌ 错误：测试mock是否存在
test('renders sidebar', () => {
  render(<Page />);
  expect(screen.getByTestId('sidebar-mock')).toBeInTheDocument();
});
```

**为什么这是错误的：**
- 你在验证mock是否工作，而不是组件是否工作
- mock存在时测试通过，不存在时失败
- 这对真实行为没有任何说明

**代码审查时的关键问题：** "我们是在测试mock的行为吗？"

**正确做法：**
```typescript
// ✅ 正确：测试真实组件或不要mock它
test('renders sidebar', () => {
  render(<Page />);  // 不要mock sidebar
  expect(screen.getByRole('navigation')).toBeInTheDocument();
});

// 或者如果必须mock sidebar以实现隔离：
// 不要对mock进行断言 - 测试Page在sidebar存在时的行为
```

### 检查函数

```
在对任何mock元素进行断言之前：
  问："我是在测试真实组件行为还是仅仅测试mock的存在？"

  如果是测试mock的存在：
    停止 - 删除该断言或取消mock该组件

  改为测试真实行为
```

## 反模式2：生产代码中的仅测试方法

**违规示例：**
```typescript
// ❌ 错误：destroy()仅在测试中使用
class Session {
  async destroy() {  // 看起来像生产API！
    await this._workspaceManager?.destroyWorkspace(this.id);
    // ... 清理
  }
}

// 在测试中
afterEach(() => session.destroy());
```

**为什么这是错误的：**
- 生产类被仅用于测试的代码污染
- 如果在生产环境中意外调用会很危险
- 违反YAGNI原则和关注点分离
- 混淆了对象生命周期与实体生命周期

**正确做法：**
```typescript
// ✅ 正确：测试工具处理测试清理
// Session没有destroy() - 在生产环境中它是无状态的

// 在test-utils/中
export async function cleanupSession(session: Session) {
  const workspace = session.getWorkspaceInfo();
  if (workspace) {
    await workspaceManager.destroyWorkspace(workspace.id);
  }
}

// 在测试中
afterEach(() => cleanupSession(session));
```

### 检查函数

```
在向生产类添加任何方法之前：
  问："这个方法是否仅被测试使用？"

  如果是：
    停止 - 不要添加它
    将它放在测试工具中

  问："这个类是否拥有这个资源的生命周期？"

  如果不是：
    停止 - 这不是该方法应该所在的类
```

## 反模式3：不理解就使用Mock

**违规示例：**
```typescript
// ❌ 错误：Mock破坏了测试逻辑
test('detects duplicate server', () => {
  // Mock阻止了测试依赖的配置写入！
  vi.mock('ToolCatalog', () => ({
    discoverAndCacheTools: vi.fn().mockResolvedValue(undefined)
  }));

  await addServer(config);
  await addServer(config);  // 应该抛出异常 - 但不会！
});
```

**为什么这是错误的：**
- 被mock的方法有测试依赖的副作用（写入配置）
- 为了"安全"而过度mock破坏了实际行为
- 测试因错误的原因通过或莫名其妙地失败

**正确做法：**
```typescript
// ✅ 正确：在正确的层级进行mock
test('detects duplicate server', () => {
  // Mock慢的部分，保留测试需要的行为
  vi.mock('MCPServerManager'); // 只mock慢的服务器启动

  await addServer(config);  // 配置已写入
  await addServer(config);  // 检测到重复 ✓
});
```

### 检查函数

```
在mock任何方法之前：
  停止 - 还不要mock

  1. 问："真实方法有什么副作用？"
  2. 问："这个测试是否依赖这些副作用？"
  3. 问："我是否完全理解这个测试需要什么？"

  如果依赖副作用：
    在更低的层级进行mock（实际的慢速/外部操作）
    或者使用保留必要行为的测试替身
    而不是测试依赖的高级方法

  如果不确定测试依赖什么：
    首先用真实实现运行测试
    观察实际需要发生什么
    然后在正确的层级添加最小化的mock

  危险信号：
    - "我会mock这个以保证安全"
    - "这可能很慢，最好mock它"
    - 在不理解依赖链的情况下进行mock
```

## 反模式4：不完整的Mock

**违规示例：**
```typescript
// ❌ 错误：部分mock - 只包含你认为需要的字段
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' }
  // 缺失：下游代码使用的元数据
};

// 后来：当代码访问response.metadata.requestId时崩溃
```

**为什么这是错误的：**
- **部分mock隐藏了结构假设** - 你只mock了你知道的字段
- **下游代码可能依赖你未包含的字段** - 静默失败
- **测试通过但集成失败** - Mock不完整，真实API完整
- **错误的信心** - 测试对真实行为没有任何证明

**铁律：** 必须mock现实中存在的完整数据结构，而不仅仅是你的测试直接使用的字段。

**正确做法：**
```typescript
// ✅ 正确：镜像真实API的完整性
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' },
  metadata: { requestId: 'req-789', timestamp: 1234567890 }
  // 真实API返回的所有字段
};
```

### 检查函数

```
在创建mock响应之前：
  检查："真实的API响应包含哪些字段？"

  行动：
    1. 从文档/示例中检查实际的API响应
    2. 包含系统在下游可能使用的所有字段
    3. 验证mock与真实响应模式完全匹配

  关键：
    如果你正在创建mock，你必须理解整个结构
    当代码依赖省略的字段时，部分mock会静默失败

  如果不确定：包含所有文档化的字段
```

## 反模式5：将集成测试作为事后补救

**违规示例：**
```
✅ 实现完成
❌ 没有编写测试
"准备测试"
```

**为什么这是错误的：**
- 测试是实现的一部分，不是可选的后续工作
- TDD本可以避免这个问题
- 没有测试就不能声称完成

**正确做法：**
```
TDD循环：
1. 编写失败的测试
2. 实现以通过测试
3. 重构
4. 然后声称完成
```

## 当Mock变得过于复杂时

**警告信号：**
- Mock设置比测试逻辑还长
- 为了让测试通过而mock所有东西
- Mock缺少真实组件拥有的方法
- 当mock改变时测试崩溃

**代码审查时的关键问题：** "我们需要在这里使用mock吗？"

**考虑：** 使用真实组件的集成测试通常比复杂的mock更简单

## TDD如何防止这些反模式

**TDD为什么有帮助：**
1. **先写测试** → 强制你思考你实际在测试什么
2. **观察它失败** → 确认测试测试的是真实行为，而不是mock
3. **最小化实现** → 不会悄悄加入仅用于测试的方法
4. **真实依赖** → 在mock之前你会看到测试实际需要什么

**如果你在测试mock行为，说明你违反了TDD** - 你在没有先看到测试在真实代码上失败的情况下就添加了mock。

## 快速参考

| 反模式 | 解决方案 |
|--------|---------|
| 对mock元素进行断言 | 测试真实组件或取消mock |
| 生产代码中的仅测试方法 | 移到测试工具中 |
| 不理解就mock | 先理解依赖，最小化mock |
| 不完整的mock | 完全镜像真实API |
| 测试作为事后补救 | TDD - 测试优先 |
| 过度复杂的mock | 考虑集成测试 |

## 危险信号

- 断言检查`*-mock`测试ID
- 方法仅在测试文件中被调用
- Mock设置占测试的>50%
- 移除mock时测试失败
- 无法解释为什么需要mock
- 为了"保证安全"而mock

## 底线

**Mock是隔离的工具，不是要测试的东西。**

**如果测试在验证mock行为 → 违反了TDD原则。**

解决方案：
1. 测试真实行为而非mock存在性
2. 质疑mock的必要性
3. 在正确的层级进行隔离
