# Info Push Android 半原生改造 Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 将现有 Info Push Web 能力迁移为 Android App（Jetpack Compose），直连现有 API，用户数据保存到设备本地。

**Architecture:** 采用 Data/Domain/UI 三层。Data 层用 Retrofit 访问现有 API，用 Room 存本地 sources/sourceItems/favorites/messages/preferences；Repository 负责“先本地展示，再远端刷新并回写本地”。UI 层用 Compose + ViewModel + StateFlow 实现首页、信息源、收藏、消息中心四个页面，并保持“收藏成功”即时反馈。

**Tech Stack:** Kotlin, Jetpack Compose, Navigation Compose, ViewModel, Coroutines/Flow, Retrofit + OkHttp + Moshi, Room, DataStore（可选偏好轻存储）, JUnit + Turbine + MockWebServer。

---

### Task 1: 创建 Android 项目骨架与依赖

**Files:**
- Create: `apps/info-push/android/app/build.gradle.kts`
- Create: `apps/info-push/android/app/src/main/AndroidManifest.xml`
- Create: `apps/info-push/android/app/src/main/java/com/infopush/app/MainActivity.kt`
- Create: `apps/info-push/android/app/src/main/java/com/infopush/app/InfoPushApp.kt`
- Modify: `apps/info-push/android/README.md`

**Step 1: Write the failing test**
- Create: `apps/info-push/android/app/src/test/java/com/infopush/app/ProjectSanityTest.kt`
```kotlin
import org.junit.Test
import kotlin.test.assertTrue

class ProjectSanityTest {
  @Test fun appModuleLoads() {
    assertTrue(true)
  }
}
```

**Step 2: Run test to verify it fails**
Run: `./gradlew :app:testDebugUnitTest`
Expected: FAIL（项目/依赖尚未完整）

**Step 3: Write minimal implementation**
- 初始化 app 模块、Compose、Retrofit、Room、Navigation 依赖
- 增加 `MainActivity` + `InfoPushApp` 最小可运行页面（空壳导航）

**Step 4: Run test to verify it passes**
Run: `./gradlew :app:testDebugUnitTest`
Expected: PASS

**Step 5: Commit**
```bash
git add apps/info-push/android
git commit -m "feat(android): 初始化 Compose + Room + Retrofit 项目骨架"
```

### Task 2: 定义 API 模型与 Retrofit 接口

**Files:**
- Create: `apps/info-push/android/app/src/main/java/com/infopush/app/data/remote/model/ApiModels.kt`
- Create: `apps/info-push/android/app/src/main/java/com/infopush/app/data/remote/InfoPushApi.kt`
- Create: `apps/info-push/android/app/src/main/java/com/infopush/app/data/remote/NetworkModule.kt`
- Test: `apps/info-push/android/app/src/test/java/com/infopush/app/data/remote/InfoPushApiTest.kt`

**Step 1: Write the failing test**
- 用 `MockWebServer` 验证 `/v1/sources/home`、`/v1/favorites`、`/v1/data/export` 反序列化

**Step 2: Run test to verify it fails**
Run: `./gradlew :app:testDebugUnitTest --tests "*InfoPushApiTest"`
Expected: FAIL（接口/模型缺失）

**Step 3: Write minimal implementation**
- 定义响应模型（sources/items/favorites/messages/preferences/export/import）
- 定义 Retrofit 接口与基础网络模块

**Step 4: Run test to verify it passes**
Run: `./gradlew :app:testDebugUnitTest --tests "*InfoPushApiTest"`
Expected: PASS

**Step 5: Commit**
```bash
git add apps/info-push/android/app/src/main/java/com/infopush/app/data/remote apps/info-push/android/app/src/test/java/com/infopush/app/data/remote
 git commit -m "feat(android): 增加 InfoPush API 接口与数据模型"
```

### Task 3: 定义 Room 实体、DAO、数据库

**Files:**
- Create: `apps/info-push/android/app/src/main/java/com/infopush/app/data/local/entity/*.kt`
- Create: `apps/info-push/android/app/src/main/java/com/infopush/app/data/local/dao/*.kt`
- Create: `apps/info-push/android/app/src/main/java/com/infopush/app/data/local/InfoPushDatabase.kt`
- Test: `apps/info-push/android/app/src/test/java/com/infopush/app/data/local/DaoTest.kt`

**Step 1: Write the failing test**
- In-memory Room 测试：insert + query + upsert + delete favorites

**Step 2: Run test to verify it fails**
Run: `./gradlew :app:testDebugUnitTest --tests "*DaoTest"`
Expected: FAIL

**Step 3: Write minimal implementation**
- 实体：`SourceEntity`, `SourceItemEntity`, `FavoriteEntity`, `MessageEntity`, `PreferenceEntity`
- DAO 提供页面所需最小查询和写入

**Step 4: Run test to verify it passes**
Run: `./gradlew :app:testDebugUnitTest --tests "*DaoTest"`
Expected: PASS

**Step 5: Commit**
```bash
git add apps/info-push/android/app/src/main/java/com/infopush/app/data/local apps/info-push/android/app/src/test/java/com/infopush/app/data/local
 git commit -m "feat(android): 增加 Room 数据库与 DAO"
```

### Task 4: Repository 与同步策略（本地优先）

**Files:**
- Create: `apps/info-push/android/app/src/main/java/com/infopush/app/data/repo/InfoPushRepository.kt`
- Create: `apps/info-push/android/app/src/main/java/com/infopush/app/domain/*.kt`
- Test: `apps/info-push/android/app/src/test/java/com/infopush/app/data/repo/InfoPushRepositoryTest.kt`

**Step 1: Write the failing test**
- 验证“先读本地，再刷新远端并回写本地”
- 验证收藏乐观更新与重复收藏处理

**Step 2: Run test to verify it fails**
Run: `./gradlew :app:testDebugUnitTest --tests "*InfoPushRepositoryTest"`
Expected: FAIL

**Step 3: Write minimal implementation**
- Repository 提供：`observeFeed`, `refreshSource`, `addFavorite`, `removeFavorite`, `observeMessages`, `importData`, `exportData`

**Step 4: Run test to verify it passes**
Run: `./gradlew :app:testDebugUnitTest --tests "*InfoPushRepositoryTest"`
Expected: PASS

**Step 5: Commit**
```bash
git add apps/info-push/android/app/src/main/java/com/infopush/app/data/repo apps/info-push/android/app/src/main/java/com/infopush/app/domain apps/info-push/android/app/src/test/java/com/infopush/app/data/repo
 git commit -m "feat(android): 实现本地优先仓库与同步策略"
```

### Task 5: Compose 页面与导航

**Files:**
- Create: `apps/info-push/android/app/src/main/java/com/infopush/app/ui/navigation/AppNav.kt`
- Create: `apps/info-push/android/app/src/main/java/com/infopush/app/ui/feed/*`
- Create: `apps/info-push/android/app/src/main/java/com/infopush/app/ui/sources/*`
- Create: `apps/info-push/android/app/src/main/java/com/infopush/app/ui/favorites/*`
- Create: `apps/info-push/android/app/src/main/java/com/infopush/app/ui/messages/*`
- Test: `apps/info-push/android/app/src/test/java/com/infopush/app/ui/FavoriteFeedbackViewModelTest.kt`

**Step 1: Write the failing test**
- ViewModel 测试：收藏成功后发出 `UiEvent.Toast("收藏成功")`，按钮状态变已收藏

**Step 2: Run test to verify it fails**
Run: `./gradlew :app:testDebugUnitTest --tests "*FavoriteFeedbackViewModelTest"`
Expected: FAIL

**Step 3: Write minimal implementation**
- 四个页面最小可用 UI
- 收藏点击后立刻更新状态 + Toast 事件

**Step 4: Run test to verify it passes**
Run: `./gradlew :app:testDebugUnitTest --tests "*FavoriteFeedbackViewModelTest"`
Expected: PASS

**Step 5: Commit**
```bash
git add apps/info-push/android/app/src/main/java/com/infopush/app/ui apps/info-push/android/app/src/test/java/com/infopush/app/ui
 git commit -m "feat(android): 完成首页/信息源/收藏/消息中心页面与收藏反馈"
```

### Task 6: 导入导出（设备迁移）

**Files:**
- Create: `apps/info-push/android/app/src/main/java/com/infopush/app/domain/ImportExportUseCase.kt`
- Modify: `apps/info-push/android/app/src/main/java/com/infopush/app/ui/settings/SettingsScreen.kt`（如不存在则创建）
- Test: `apps/info-push/android/app/src/test/java/com/infopush/app/domain/ImportExportUseCaseTest.kt`

**Step 1: Write the failing test**
- 导出生成 JSON（含 version + exportedAt + data）
- 导入支持 replace/merge

**Step 2: Run test to verify it fails**
Run: `./gradlew :app:testDebugUnitTest --tests "*ImportExportUseCaseTest"`
Expected: FAIL

**Step 3: Write minimal implementation**
- 实现 JSON 导入导出（本地文件）
- UI 增加“导出数据”“导入数据（replace/merge）”入口

**Step 4: Run test to verify it passes**
Run: `./gradlew :app:testDebugUnitTest --tests "*ImportExportUseCaseTest"`
Expected: PASS

**Step 5: Commit**
```bash
git add apps/info-push/android/app/src/main/java/com/infopush/app/domain apps/info-push/android/app/src/main/java/com/infopush/app/ui/settings apps/info-push/android/app/src/test/java/com/infopush/app/domain
 git commit -m "feat(android): 增加本地导入导出能力用于设备迁移"
```

### Task 7: 验证与文档

**Files:**
- Modify: `apps/info-push/android/README.md`
- Create: `apps/info-push/android/docs/acceptance.md`

**Step 1: Write the failing test**
- 无新增功能测试；先执行全量验证命令

**Step 2: Run test to verify current status**
Run: `./gradlew :app:testDebugUnitTest`
Expected: PASS

**Step 3: Write minimal implementation**
- README 增加运行方式、环境变量（API baseUrl）、导入导出说明
- 验收文档写明四个页面与关键交互

**Step 4: Run test to verify it passes**
Run: `./gradlew :app:testDebugUnitTest`
Expected: PASS

**Step 5: Commit**
```bash
git add apps/info-push/android/README.md apps/info-push/android/docs/acceptance.md
 git commit -m "docs(android): 补充运行说明与验收文档"
```
