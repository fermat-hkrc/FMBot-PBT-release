# ArkTS 签名与设备判定

[English](arkts-device.md) · [命令参考](subcommands.zh.md) · [安装指南](installation.zh.md)

## 源码测试任务

```bash
fmbot-pbt build-run --build-cmd "<构建命令>" --target <源码文件> --func <函数> \
  --sign-apl system_basic --sign-feature hos_system_app --result-source auto
```

`build-run` 与 `/skill:pbt-build-run` 共用以下参数、校验和执行：

| 参数 | 说明 |
|---|---|
| `--result-source auto\|stdout\|hilog` | 默认 `auto`，接受完整 stdout 或关联本次运行的设备结果 |
| `--sign-apl normal\|system_basic` | SDK 调试 profile 的 APL，默认 `normal` |
| `--sign-feature hos_normal_app\|hos_system_app` | 匹配的应用特性，省略时由 APL 决定 |
| `--sign-profile normal\|system` | 对应 APL/feature 组合的简写，冲突组合会被拒绝 |
| `--device-target <hdc>` | 选择设备 |
| `--test-bundle`、`--test-module`、`--test-class` | 显式选择包、模块和 `Suite#case` |
| `--ability-grace-ms`、`--test-ability` | 启动恢复等待时间（20000，0 禁用）与 Ability 名（`TestAbility`） |
| `--idle-timeout-ms <ms>` | 未观察到测试进展时的等待上限（300000，0 禁用此上限） |
| `--hap <entry.hap>`、`--test-hap <test.hap>` | 下述已有产物模式，不生成新性质 |

引擎通过 `PBT_ARKTS_DEVICE_OPTIONS`（JSON）把配置传给工具，优先使用 CLI 参数，不要手工设置该变量。
`HDC`、`JAVA` 可选择已准备好的可执行文件。
源码测试任务按 `build-profile.json5` 确认源码所属模块。如果给定的 `ohosTest` 构建命令选择了依赖该库的 HAP，就在该 HAP 中注册测试并导入库的原始导出，保留命令中完整的模块与目标列表。重新构建、签名最新源码与测试后，用 `fmbot-pbt arkts-test --class <PbtSuite> --test-hap <current-signed-ohosTest.hap>`（或选择 `Suite#case`）验证，不得默认执行全部单元测试。包参数用于此验证命令，不用于初始 `build-run`，后者会因此选择已有产物模式。运行器安装并记录指定测试 HAP 的指纹；HAP 早于当前源码、测试输入时，结果为不确定。保留该文件和源码树以供离线收尾。通过核实的包导出或源码路径导入真实被测声明，禁止复制实现。
CLI 和会话内准备流程通过 `~/tmp/fmbot-pbt-campaign-locks/` 持有工作区及输出目录；重叠任务在构建或清理前退出 2 并给出持有者信息，独立工作区可并行。只自动回收已确认退出的宿主进程，不抢占存活、无权查询或其他主机的持有者。

## 失败记录与收尾

调用方引用支持 Windows 路径、空格和限定方法名。UI 回调（`onPaste`、`onClick`、TextInput 的 `onChange`/`onSubmit`）及 `@Entry`/`@Component` 的 build 方法可作为应用入口，但引用代码必须实际调用目标。只有排除实际反例的转换才能证明不可达；去除逗号不会排除 `'1E'`。
先记录失败性质，再按随包示例调用 `record_bug`。同一问题累计拒绝 3 次、全部未解决问题累计拒绝 12 次，或仍有待修正项且 5 分钟内未解决任何一项时，以不确定结果退出 2。计时从首次拒绝开始，仅在一项修正通过且仍有其他待修正项时重新计时；无关记录通过或重复提交已通过的修正不会延长时限。`rejected-records.jsonl` 保留请求、拒绝原因和原证据；`recording-stop.json` 与 `interruption.json` 说明停止原因。不得仅为通过门禁而撤回真实失败或退役性质。
无界面 Pi `build-run` 测试整个仓库且未指定改动范围时，如果各次拒绝均指向同一条已保存的问题，第 3 次拒绝可暂停该问题的修正，继续无关测试。适用情形包括已记录反例的失败性质，或摘要仅因一条通过性质的备注描述了由失败改为通过而被拒绝。性质、备注和证据原样保留。检查点中的未解决条目阻止交付，`finalize` 也无法通过；任务保留部分结果并退出 2。如需验证受影响的工作，请使用独立输出目录启动新任务。
关联本次运行的设备失败自动保留在 `report.json.deviceFailures` 与 `findings-unrecorded.json`，即使性质或缺陷记录被拒也不会丢失。即使后续已验证的重跑解决了原调用的不确定状态，超时前记录的失败仍须处置。这些是观测，并不自动认定为产品缺陷；用 `deviceObservationIds` 关联已校验的性质。修正测试须保留失败历史并重新验证，不得静默退役；相对性质路径按已记录的仓库根目录解析。
`reference` 判定须在设计获批前给出独立的约定证据（规范、注释或调用方断言），不能把实现自身的键表或正则当成参考答案。
执行工具前保存 `campaign-state.json`。父进程退出且没有 `interruption.json` 时，`fmbot-pbt resume --out <out>` 使用该检查点对应的会话，保留范围、模型和证据起始时间。空套件或未匹配的类写入不确定结果及中断记录，退出 2，不会把零测试当成成功。
证据已记录完整但 agent 中止时，先结束活跃任务，再用 `fmbot-pbt finalize --out <out>` 校验回执并离线收尾。它保留中断历史、完成匹配检查点，不请求模型或设备。独占所有权后可将较早且已遗弃的缺失回执记为中止，但绝不记为通过；矛盾回执和未处置的失败仍会阻止完成，详见[恢复与离线收尾](subcommands.zh.md#恢复与离线收尾)。
无界面运行在 SDK 确认无待处理工作后执行最终交付检查并退出；仅有 `REPORT.md` 文件不代表完成。交互及 RPC 会话保持可用。

## 本地签名

```bash
fmbot-pbt arkts-sign app-unsigned.hap app-signed.hap --bundle com.example.app --udid DEVICE_UDID \
  --sign-apl system_basic --sign-feature hos_system_app
```

默认是 `normal`/`hos_normal_app`，`--sign-profile system` 选择系统组合。
本地 SDK 调试 profile 保留 UDID，并合并请求权限、已有 ACL 与可重复的 `--acl` 值。
不改变输入 HAP 或 SUT 签名脚本，不请求 CPKI，也不会失败后换用其他签名策略。

设备必须信任 SDK 调试证书；这不能授予零售设备权限，也不能补齐缺失的系统 SDK API。
错误 `202` 仍可能要求更换设备、证书、权限或 SDK。

SDK 优先级：`--sdk`、`local.properties` 的 `sdk.dir`、`DEVECO_SDK_HOME`、`DEVECO_HOME`/标准 DevEco 位置。
Windows 发现包含 `Program Files/Huawei/DevEco Studio/sdk`；打印最终路径，在项目外签名时使用 `--project <hap-root>`。
Java 来自 `JAVA`、发现的 DevEco `jbr` 或 PATH。读取 HAP 元数据需要 `unzip` 或支持 ZIP 的 `tar`（Windows 自带）。

## 设备结果

```bash
fmbot-pbt arkts-test --target DEVICE --bundle com.example.app --module entry_test \
  --class DisplayPbt --test-hap current-signed-ohosTest.hap --result-source auto --out pbt-out/device
```

二进制使用内置运行时，不调用模型；`sign-hap.sh`、`aa-test.sh` 脚本适配器通过 Node 调用同一实现。
`auto` 接受完整的 stdout 结果（包括已验证的原生完成报告），或能关联本次运行的新 hilog 摘要。`stdout` 不采集 hilog；`hilog` 仍要求设备日志中的摘要。
运行器在设备端过滤 hilog，避免无关历史日志阻塞测试记录。收到可用摘要且 `aa test` 结束后，运行器在原有总截止时间内短暂等待关联的 hilog 记录。到达截止时间时，会核对已按时收到的记录（包括冲突摘要），不再等待新记录。
不会清空设备日志，也不会接受部分或仅含状态的记录。

新测试 runner 在 Hypium 启动前输出 UUID 标记，随包命令卡说明写法，隔离 runner 已自带。
同一 PID 出现其他运行的标记后，不再关联后续记录，但保留该边界之前已归属的证据。
旧的预签名 runner 可改用已核实的新包进程 PID 关联；缺少或有歧义时仍为不确定，即使其他日志里出现完整摘要。

| 退出码 | 说明 |
|---|---|
| `0` | 完整、数量一致且有实际执行的摘要，无失败或错误 |
| `1` | 完整摘要中有真实失败或错误 |
| `2` | 无完整结果、结果冲突/无关联/无效、传输失败或停止推进 |

`--class Suite#case` 选择用例；嵌套套件使用 Hypium 完整路径，例如 `Outer.Inner#case`。`--timeout-ms` 为单用例上限（默认 120000），`--run-timeout-ms` 为测试等待上限（1800000）。带 Hypium 可选超时、标签参数的用例，以及直接定义在 `List.test.ets` 中的用例，同样需要执行回执。仅用于注册其他套件的索引本身不能作为性质的测试文件。
测试任务内（`PBT_OUT_DIR` 已设置），所有入口均把 `stdout.log`、新的相关 `hilog.log`、`diagnostic.log` 和 `result.json` 写到 `<out>/device-run/<run-id>/`，不采用辅助命令的 `--out`。结果含 Hypium 套件、用例的完成状态，源码、测试、注册文件及 HAP 的 SHA-256 指纹，ISO `startedAt`/`endedAt` 和失败观测。报告中每个测试文件的所有用例均须有匹配的新回执；仅当输入未变时，分次选择用例的结果才能累计。`List.test.ets` 导出的同步 `testsuite()` 须沿可达路径调用默认套件。死分支注册、类选择参数或无关冒烟摘要不能清除待执行的性质；缺少用例或产物证据时，交付仍退出 2。启动前将调用登记到 `device-run/runs.jsonl`。任务外的独立调用仍写入 `--out`，否则用新的 `~/tmp/pbt-aa-*` 目录。

### 不依赖 hilog 的具名结果

生成的隔离项目自动安装仅供测试使用的报告器。已有 Hypium 1.0.24/1.0.25 源码测试模块可从随包 `pbt-build-run` skill 复制 `references/arkts/runner/FmbotCaseReporter.js`，放在 `src/ohosTest/ets/test/List.test.ets` 旁。在该文件中从 `./FmbotCaseReporter` 导入 `{ installFmbotCaseReporter }`，并在导出的 `testsuite()` 中、注册套件前调用。重新构建并签名测试 HAP 一次；不得修改产品代码或改写传入的预签名包。

报告器在 Hypium 的终态回调中记录每个用例的名称，以及通过、失败或错误状态。小报告直接放在原生 `aa test` 完成消息中，不需要 hilog。较大报告的完成消息只含计数；具名记录按页写入先前的 `aa` 输出，每条记录不会跨页。启用 hilog 采集时还有带标签的镜像。宿主核对运行 UUID 和本次调用的新票据，相同镜像只计一次，并在 `result.json` 旁将收到的结果保存为 `case-report.json`。

即使总数有效，丢失的分页或名称仍会留下 `namedCaseGap`。空格和水平制表符是套件名、用例名的一部分。名称含其他控制字符或换行时，无法可靠比对文本日志；运行器保留回调记录并退出 `2`。计数冲突（包括零用例或全部忽略的摘要）、具名结果矛盾或完成消息格式无效时，也退出 `2`。没有报告器的旧 HAP 仍使用 stdout/hilog，无法恢复两个通道都未保留的名称。

### 旧测试包的具名回执缺口

类运行摘要通过并不等于每个用例都已按名称确认。如果有效摘要中的非忽略结果多于 `result.json.cases` 已记录的结果，运行器会给出警告，并在 `namedCaseGap` 中分别写入 `unnamedPasses` 和 `unnamedFailuresOrErrors`。例如，摘要显示 7 个通过但仅有 5 个具名完成记录时，`unnamedPasses` 为 2，这两个用例仍未验证。退出码表示套件摘要的结果，不表示具名证据完整。交付检查按套件、测试产物和原因归类回执缺口，明细列出准确的用例标识及受影响的性质 ID。复制完整的 `arkts-test --class 'Suite#case'` 命令行，只执行该用例。保留 `PBT_OUT_DIR` 以将回执写入原任务；若命令中有 `PBT_SCOPE_FILE`，也应保留，以便在记录性质结果之前为指定源码计算指纹。Windows 下使用 PowerShell。若 HAP 或目标不明或有歧义，先核实并替换占位符。确认已安装匹配的应用 HAP，或通过 `--hap` 提供已验证的文件。选择其他候选产物前重新检查交付条件。等正在执行的调用结束，保留其原始日志及回执；定向执行失败时应处置失败，不要为得到通过而重跑。这条诊断不会自动发起测试。

如需自动补齐，请在启动无界面 Pi ArkTS 源码 `build-run` 时显式指定 `--recover-missing-cases`。[从终端启动](subcommands.zh.md#build-run)无需重定向标准输入，但构建命令不得等待终端输入。任务会提示在首次运行性质套件的 `arkts-test` 命令中同时传入两个签名包（`--hap` 和 `--test-hap`），不要将包参数传给 `build-run`，并保持两个包不变直至收尾。通过的类回执须能唯一确认这两个包及设备。缺任一包时仅提示手动处理。

模型结束后，worker 可依次执行符合条件的缺失用例。`--recover-max-cases N` 限制符合条件的尝试次数（正整数，默认 `3`，须搭配 `--recover-missing-cases`）。路径缺失、有歧义或无法重现执行快照的用例不占预算，也不阻止其他符合条件的用例运行。时间限制不变：单次测试等待上限两分钟，含准备的总预算五分钟。到达截止时间后可单独进行一次有超时上限的设备状态检查，确认应用已退出才释放锁。

worker 保持任务租约，逐次申请正常设备锁但不回收旧锁，并重新核对输入、具名执行回执及全部交付门禁。未指定恢复配方时只使用当前的部分类回执，不重建或签名。有其他交付阻塞时不启动补齐；执行路径或输入变化、结果失败或不确定、验证无进展时停止。不会再调用模型，也不会重试不确定或失败的测试。若仍有回执缺口，交付未完成（退出 `2`），回执和手动命令会保留。自动补齐默认关闭，不适用于 `--no-auto-resume`、TUI、会话内 skill、ACP、隔离或预签名执行。

### 测试编辑后的恢复

在 `--recover-missing-cases` 之外另传 `--case-recovery-recipe recovery.json`，可授权收尾阶段尝试构建并签名一次。输出目录需位于支持硬链接的文件系统中，worker 才能发布完整的构建凭据而不覆盖已有证据。例如，项目专用配方可写成：

```json
{
  "command": "bash ./rebuild-and-sign.sh",
  "inputs": ["rebuild-and-sign.sh", "AppScope", "shared/src"]
}
```

监督进程在启动时读取文件，不从模型、回执或保存的任务状态获取命令。命令在 ArkTS 项目内的 `--build-dir` 执行，POSIX 使用 Bash，Windows 使用 `cmd.exe`，不读取标准输入。使用可信的宿主配方，按当前输入重建已注册的测试模块及其应用，等待全部构建和签名工作结束，将两个新签名包分别写到 `PBT_CASE_RECOVERY_APP_HAP` 和 `PBT_CASE_RECOVERY_TEST_HAP` 指定的新路径。使用匹配的签名策略，不得重新签署旧测试字节码、修改原包、安装到设备或执行测试。

`inputs` 列出配方的额外构建输入，可用相对于 `--build-dir` 的路径或绝对路径，支持文件与目录。请列出配方使用的所有其他依赖、资源、脚本与构建签名配置，不要包含输出或生成缓存。worker 还会对测试模块的 `src`、报告引用的源码及已知项目/模块构建文件计算快照。不可读或有别名的输入会阻止恢复。两个输出须与原模块身份一致，测试字节码须有变化，每份新回执均要求输入未变且设备安装成功。

worker 先补齐有当前有效路径且符合条件的用例。若尝试次数仍有余量，同一次收尾可调用已授权的配方，处理没有当前路径的缺口。失效缺口须指向同一个测试模块，并有本次任务中无歧义的通过类回执、可校验的原应用包与测试包，以及仍注册的套件或其注册文件的改动。两条路径共享尝试次数上限和五分钟总预算，构建也计入总时限。构建后重新核对全部交付条件；每个缺失用例（包括使用旧包通过过的用例）都需要新的精确回执。worker 在构建前后及每个重建用例通过后，重新计算刚补齐的当前路径的输入快照；即使已记录文件未变，新增输入也会阻止收尾。输入再变则停止，不作第二次构建。日志、输入快照及新包保留在 `<out>/case-recovery-build-*/`，须连同回执保留以供离线校验。首份通过回执验证新包后，运行时诊断与 `finalize` 会为计划中剩余的用例提供两个包的原始路径及设备，即使 worker 尚未执行该套件。这些提示不算执行证据。请使用两个原始路径，不要复制。运行器会再次检查构建输入，包括构建时不存在的配置文件；既往失败和回执也保留。普通 `resume` 和离线 `finalize` 不会执行配方。

### 排查设备结果

若恢复构建的包缺少 `build-artifact.json` 或该文件不可读，请重新构建并签名，不要修改构建凭据。损坏的构建记录不会阻止无关 HAP 的运行。

排查回执缺口时，先保留 `stdout.log`、`hilog.log`，共享前对副本脱敏。将具名完成记录与回执比对，仅使用此次运行的 stdout 和可归属的 hilog 记录（UUID 标记，或经核实的新鲜 PID 记录）；`hilog.log` 也可能包含其他进程的日志。区分缺少终态记录和解析遗漏，不得凭其他进程的记录、选择参数或总数补齐。

等待时间从启动 `aa test` 起算，不包括安装。新的运行标记、Hypium 用例状态、递增的用例序号和完整摘要可重新计时；重复或转发的事件、倒计时和无关日志不会重新计时。若测试长时间没有可观察的进展属于正常情况，请增大 `--idle-timeout-ms`；设为 `0` 时仅保留总截止时间。进展不延长 `--run-timeout-ms`。

输出暂不可见时，按 `device-run/runs.jsonl` 中的运行 ID 读取对应的 `progress.json`。它记录最近一次进展（也可能没有）、截至 `updatedAt` 已等待的毫秒数和证据目录。有新进展或定期输出状态时会更新此文件。等待超限时保留日志、记录 `reason: stalled` 并退出 2，不算测试失败。不会重试，也不能据此抢占仍有持有者或状态未知的设备。

明确请求普通重跑时，可自动关联唯一的 `empty-selection`。运行器同时检查任务输出目录和其 `pbt-out` 子目录中的登记。即使用例未完成，只要已经开始执行就不能认定为空选择。两次登记须属于同一个运行中任务的检查点，目标、包、模块和单一 `Suite` 或 `Suite#case` 选择器均相同。原调用已结束，当时设备包进程已退出，本机持有者也已退出；保留的 stdout、hilog 须均可读且无失败证据。再次执行时须安装指纹相同的测试 HAP，所有输入未变，且每个终态计数都对应所选范围内已通过的用例。仅退出 0 而无用例记录不能恢复。只在**新回执**中写入 `supersedesRun`；这不证明尚未执行的性质，也不会自动发起测试。

证明不足或有多个匹配登记时，先检查原始证据，切勿编辑回执。修正不确定的调用后，必要时用 `--supersedes-run <旧运行 ID>` 再次执行。只有目标、包、模块及已记录的类选择均相同的非空、已验证运行才能解决该次调用；原结果保留，真实失败不能被覆盖。新索引记录 `testClass`；登记后可捕获的准备失败会在输出可写时原子写入不确定回执。新索引同时记录宿主持有者。进程被强杀仍可能没有回执；离线 `finalize` 在独占所有权、已记录持有者确认退出、不存在设备锁且保留日志没有失败证据时，可将超过 30 分钟的缺失调用记为中止（旧索引可能没有持有者）。不裁剪索引，也不伪造已验证结果。目标缺失或已解决的 supersede 链接只保留警告；身份不匹配与真实失败目标仍为错误。不会自动重试。
若 Hypium 失败用例没有匹配的 PandaCheck 明细，`failures` 会记录实际观测到的 `Suite#case`，并引用回执旁 `stdout.log` 或 `hilog.log` 中的状态。无法归属的剩余数量合并为一条 `testCase: null` 的观测；请求的 `--class` 不算证据。未输出的 seed 和反例保持 `null`。同一次用例执行产生多条反例时，失败用例数只计一次；已通过用例中的反例会保留，但不计入失败用例数。每条观测仍须处置。仅有 `PANDACHECK_RESULT` 不是套件判定。

等待 `--ability-grace-ms` 后，仅当 `aa test` 尚未结束、无本次测试活动，且成功且可识别的 Ability dump 证明目标未启动时，最多执行一次外部 `aa start -a <ability> -b <bundle> -m <module>`。已有活动或 dump 状态未知时不恢复，仍受原截止时间约束；绝不启动第二次 `aa test`。

`~/tmp/fmbot-pbt-device-locks/` 下的单目标锁阻止本机重叠运行，提示 PID、存活状态、开始时间、命令及输出。
超时/中断后应用仍存活或状态未知时保留锁；检查准确的 `owner.json` 获取原运行 ID/输出。不得杀死 fmbot-pbt/hdc 或手工删除锁：进程列表可能包含本次 runner 及恢复父进程。任务上下文提供当前 PID 和租约路径，常见自杀命令会被阻止并给出修正提示。
后续明确请求的调用中，共享 runner 仅在锁超过 30 分钟、宿主持有者已退出且可识别的设备进程列表证明原包不存在时归档旧锁。竞争恢复会串行化，归档保留原持有者。原调用属于同一任务/选择且是不确定结果时，只有其回执及保留的 stdout/hilog 均无失败证据，新调用才自动关联它。缺失回执不能绕过该检查，日志不可读也阻止自动关联。只有非空的已验证结果才能解决已关联的原调用。`device-locked` 回执表示未执行，不累积交付债务，也不计为已验证执行。存活、未知及其他主机的持有者仍受保护。不要自动重试不确定或失败的测试；跨主机仍需外部设备调度。

## 已签名产物

```bash
fmbot-pbt build-run --hap app-signed.hap --test-hap test-signed.hap --device-target DEVICE --out pbt-out
# 直接调用确定性 runner：
fmbot-pbt arkts-test --hap app-signed.hap --test-hap test-signed.hap --target DEVICE --out pbt-out/device
```

任一包参数都会选择已有套件执行：原样安装指定文件并检查 SHA-256。
不执行构建命令（即使给了）、不签名、不调用模型、不编写性质；仅在相应模块已安装时省略该包。
HAP 元数据提供包名/模块名默认值；只有一个连接设备时自动选取，否则必须给目标。不能与 `--isolate` 同用。

此模式不会声称新源码或 `--target` 改动已进入旧二进制，也不会伪造测试任务的 `REPORT.md`。
`build-run` 证据位于 `<out>/device-run/<run-id>/result.json` 及日志，CLI 退出码按本次运行 receipt 校验为 `0/1/2`。
需要新性质时省略包参数，执行常规源码测试任务。

会话内适配器在准备阶段流式执行，返回 `started: false`（没有排入模型工作流）、套件退出码及证据路径，`campaign_result` 返回同一码。
两个入口共用实现；源码测试任务的验证须满足上述用例、产物回执要求。
