## 测试类型

测试策略，包含单元测试和集成测试两种测试类型：

### 单元测试

单元测试位于`test/`目录下，针对每个Widget组件和BLoC进行独立测试：

- **UI测试**：验证组件渲染、布局和交互行为
  - `test/app_test.dart`：测试CounterApp主应用组件
  - `test/counter/view/counter_page_test.dart`：测试CounterPage页面组件
  - `test/counter/view/counter_view_test.dart`：测试CounterView视图组件

- **BLoC测试**：验证业务逻辑和状态管理
  - `test/counter/cubit/counter_cubit_test.dart`：测试Counter Cubit的初始状态和状态转换

运行单元测试的方式：
```bash
# 运行所有单元测试
flutter test

# 运行特定测试文件
flutter test test/counter/cubit/counter_cubit_test.dart
```

也可以通过VS Code Flutter插件运行测试：
1. 打开测试文件
2. 点击测试方法旁边的"Run Test"图标
3. 或在VS Code底部栏选择"Run All Tests"

### 集成测试

集成测试针对实际设备或模拟器进行端到端测试，包含两种类型：

1. **Integration Test**：位于`integration_test/`目录下
   - 需要添加依赖：`flutter pub add --dev integration_test:{"sdk":"flutter"}`
   - 测试应用的完整流程，包括状态变化和UI响应

2. **Test Driver**：位于`test_driver/`目录下
   - 用于在各种平台上运行集成测试

运行集成测试命令：
```bash
# 在连接的设备上运行
flutter test integration_test/

# 在特定平台上运行（如Chrome）
flutter drive \
  --driver=test_driver/integration_test.dart \
  --target=integration_test/app_test.dart \
  -d chrome
```