# 关于本分支

`zhihulite` 分支是 Material Components for Android 的**第三方分支**，
与 Google 官方无关，也未获官方背书。上游代码与官方 README 保持原样，
本文件只描述本分支的增量。
以下描述的 `VERSION` 为本分支发布版本，当前本分支发布版本为 **1.14.0**。

## 鸣谢

预测性返回动画（predictive back）的实现来自
[@ZL114514](https://github.com/ZL114514) 的
[material-components-android](https://github.com/ZL114514/material-components-android) 分支。
本分支把这部分改动重新落在上游最新基线上，并未改变其实现思路。

## 本分支做了什么

### 1. 让 MaterialContainerTransform 支持预测性返回动画

Android 14（API 34）起，系统的返回手势可以「预览」目标页面 ——
用户把手指停在半途时，转场动画也应当停在半途，松手才决定是完成还是回弹。
这要求转场能被**任意进度驱动**，而不只是从头播到尾。

上游的 `MaterialContainerTransform` 不支持这种驱动方式，接入预测性返回后
容器转场会直接跳变而非跟手。本分支用三处改动解决它：

| 改动 | 作用 |
|---|---|
| `isSeekingSupported()` 返回 `true` | 向 AndroidX Transition 声明本转场可被 seek，这是预测性返回生效的开关 |
| 起止 View 改用 bitmap 快照（`copyViewImage`） | seek 会反复来回改变进度，直接操作活的 View 会因布局/绘制状态被反复改写而出现残影与闪烁；快照让每帧只是在画一张不变的图 |
| 监听器从 `TransitionListenerAdapter` 换到 animator 上的 `AnimatorListenerAdapter` | seek 驱动下 transition 级别的 start/end 回调时机不再可靠，改听 animator 自身的生命周期 |

配套在 `TransitionUtils` 中新增：

- `copyViewImage(sceneRoot, view, parent)` —— 把 View 截成 bitmap 并包装成
  `ImageView`；超过 1 MB 的位图会等比缩小，避免大页面截图时的内存尖峰
- `tryTransformMatrixToGlobal` / `tryTransformMatrixToLocal` —— 坐标变换辅助，
  移植自 AndroidX Transition 的对应实现

### 2. 顺带的性能收益

bitmap 快照带来的一个副作用是：转场过程中不再需要对起止 View 做重复的测量与绘制，
每帧只是把一张已有位图按矩阵画出来。页面层级越深、View 越多，这个差别越明显。

> 这条我们没有在本仓库给出量化数据。若要作为性能结论引用，
> 请自行在目标设备上做 A/B 实测（同一 APK、开关本分支的 material 依赖），
> 并同时看总时长与最大帧耗时 —— 只看总时长会漏掉把成本搬到某一帧的情况。

## 怎么用

制品发布在 [zhihulite/maven-repository](https://github.com/zhihulite/maven-repository)。

### settings.gradle

```groovy
dependencyResolutionManagement {
    repositories {
        maven {
            name = 'zhihulite-releases'
            url = 'https://raw.githubusercontent.com/zhihulite/maven-repository/main/repository/releases'
            content {
                // 只拦 material 这一个模块，其余 com.google.* 仍走 google()
                includeModule 'com.google.android.material', 'material'
            }
        }
        google()
        mavenCentral()
    }
}
```

注意必须用 `includeModule` 而非 `includeGroup 'com.google.android.material'`：
后者会把该组下所有模块都拦到本仓库，而本仓库只有 `material` 一个，
其余（如 `material-icons`）会解析失败。

### build.gradle

```groovy
dependencies {
    implementation 'com.google.android.material:material:VERSION'
}
```

### 确认你用的是本分支的制品

坐标和官方完全一样，同一个 classpath 上**只能有一个 material 生效**。
如果你的工程里有别的依赖间接引入了官方 material（AndroidX 组件常有这种传递依赖），
有可能会用错版本。稳妥起见可以加约束：

```groovy
dependencies {
    implementation 'com.google.android.material:material:VERSION'
    constraints {
        implementation('com.google.android.material:material:VERSION') {
            because '第三方分支：覆盖任何传递引入的官方 material'
        }
    }
}
```

验证最终选中的版本：

```bash
./gradlew :app:dependencyInsight --configuration debugRuntimeClasspath \
    --dependency com.google.android.material:material
```

输出里出现 `By constraint: 第三方分支：…`、版本号为 VERSION，就说明约束生效了。

不过这个命令**不显示制品来自哪个仓库**，版本号相同也不代表字节相同。
想确证拿到的是本分支的产物，可以比对 AAR 的 sha1：

```bash
# 本分支构建产物
sha1sum lib/build/outputs/aar/lib-release.aar
# 消费方缓存中实际用的那份
find ~/.gradle/caches/modules-2/files-2.1/com.google.android.material \
    -name 'material-VERSION.aar' -exec sha1sum {} +
```

### 切回官方版本

本分支**没有新增任何公开 API**，也没有改过现有方法的签名：

- `TransitionUtils` 是包级私有类（`class TransitionUtils`，无 `public` 修饰），
  新增的 `copyViewImage` 等方法虽然标了 `public static`，但出不了
  `com.google.android.material.transition` 包
- `isSeekingSupported()` 是对 AndroidX Transition 既有方法的 `@Override`，
  不是新 API

所以换回官方制品只需要改依赖坐标：

```groovy
implementation 'com.google.android.material:material:VERSION'
```

再把 `settings.gradle` 里的 `zhihulite-releases` 仓库块删掉（至少删掉其中的
`includeModule` 那行）。业务代码完全不用动 —— 唯一的行为差异是
`MaterialContainerTransform` 不再跟随预测性返回手势，会退回跳变式的转场。

## 从源码构建与发布

```bash
# 构建
./gradlew :lib:assembleRelease

# 发布（默认写到 ../maven-repository/repository/releases）
./gradlew :lib:publishReleasePublicationToMavenRepository

# 换个发布位置
./gradlew :lib:publishReleasePublicationToMavenRepository -PmavenRepoUrl=<目标路径>
```

只发布 `:lib`。`:catalog`（示例 app）和 `:testing` 没开 `maven-publish`，不会被发出去。
版本号改 `build.gradle` 里的 `mdcLibraryVersion`。

发布只是往目标目录写文件，后续需要由制品仓库那边 `git commit` + `git push` 完成分发。

### 和上游构建配置的差异

- `mavenRepoUrl` 默认值从 `/tmp/myRepo/` 改成了并排的 `maven-repository`；
  原来的默认值在 Windows 上会落到盘符根的 tmp 目录，而且不带 releases/snapshots 分区。
- POM 的 `url`/`scm` 指向本仓库而非上游 —— 制品带的是本分支的改动，
  使用者顺着上游 URL 找不到对应的源码。

## 许可

Apache-2.0，和上游一样，见 [LICENSE](LICENSE)。本分支不改变许可条款。