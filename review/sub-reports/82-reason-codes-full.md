# 原因码全表（`fb75afc`）

列说明：端＝设备端（Kotlin）、主机端（Python）或两端；处数＝直接构造处数（括号里是别处构造或动态构造的位置数）；结果＝通常的主结果与退出码（2 未发出，3 结果未知，1 失败，2/3 按是否已发出，— 不作为主错误）；类别见报告 §1；受众：C 调用者可见，U 只在设备管理界面，I 内部，D 诊断字段；运行＝37 份真实运行原件里出现次数（含旧码名的基线运行另计在报告里）。

| 码 | 端 | 处数 | 结果 | 类别 | 受众 | 指引点名 | 按码分支 | 运行 | 说明 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `actionLabel` | 设备 | 2 | 2 | N1 | C |  |  |  | 改用 --action-id；候选并入 arguments |
| `actionName` | 设备 | 1 | 2 | N1 | C |  |  |  | 同上 |
| `actionProtocol` | 设备 | 2（+1） | 3 | N7 | C |  |  |  | 动作回调布局异常，动作可能已发出 |
| `actionUnavailable` | 设备 | 4 | 2 | N3 | C |  |  |  | 混合：换动作编号／改用 click／改用别的输入 |
| `actionUnsupported` | 设备 | 1 | 2 | N3 | C |  |  |  | 改用专用命令 |
| `activityUnavailable` | 设备 | 1 | 2 | N8 | C |  |  |  | 系统服务不可用 |
| `ambiguous` | 设备 | 2 | 2 | N2 | C |  |  |  | 从列出的候选里选具体引用或补选择条件 |
| `apkUnavailable` | 主机 | 1 | 2 | N6 | C | reference.md:36 |  |  | 源码安装没有附带 APK |
| `appNotLaunchable` | 设备 | 1 | 2 | N3 | C |  |  |  | 包没有启动入口 |
| `arguments` | 两端 | 216 | 2 | N1 | C | reference.md:84 | UiRun.kt:129（argument=condition） | 1 | 通用参数错误；带 argument 的取值 23 种 |
| `artifact` | 设备 | 1 | 2 | N8 | C |  |  |  | 内部请求不一致 |
| `backendCache` | 设备 | 1 | 2 | N8 | C |  |  |  | 推断：无障碍缓存配置失败 |
| `backendConnection` | 设备 | 2 | 2/3 | N4 | C |  |  |  | 推断：界面服务连接断开，下次命令重连 |
| `busy` | 两端 | 13 | 2 | N4 | C | SKILL.md:23；connection.md:93 |  |  | 混合：等占用者结束／查询文件步骤编号 |
| `cancelled` | 两端 | 3 | 1 | N4 | C |  |  |  | 传输被取消或到期 |
| `captureMismatch` | 主机 | 11（+1） | 2/3 | N8 | C |  |  |  | 主机校验附件不符 |
| `captureUnavailable` | 两端 | 4 | 2 | N2 | C |  |  |  | 采集已不是当前的，重新 snapshot |
| `certificate` | 主机 | 4 | 2 | N6 | C |  |  |  | 证书文件或内容无效 |
| `certificateClock` | 设备 | 1 | - | N6 | U |  |  |  | 推断：设备端网关启动时报给管理界面 |
| `certificateExpired` | 设备 | 1 | - | N6 | U |  |  |  | 同上 |
| `changed` | 设备 | 8 | 2 | N2 | C | reference.md:84 | UiRefRepair.kt:51（subject=selection） |  | subject=container/selection；subject：container/selection |
| `checkPending` | 设备 | 0（+1） | - | N9 | D |  |  |  | 滚动结果里的诊断字段 |
| `chunk` | 两端 | 3（+1） | 1 | N8 | C |  |  |  | 传输完整性 |
| `clipboardTransfer` | 设备 | 1 | 2 | N3 | C |  |  |  | 原剪贴板含 URI，借用粘贴不可用 |
| `clipboardUnavailable` | 设备 | 2 | 2 | N3 | C |  |  |  | 混合：剪贴板服务不可用／原内容读不到 |
| `clipboardUnconfirmed` | 设备 | 1 | 2 | N3 | C |  |  |  | 借用写入未确认，未发 PASTE |
| `collectionLimit` | 设备 | 0（+3） | - | N9 | D |  | UiPublication.kt:204 |  | 采集时限缺口标记，也是数据词汇 |
| `collectionProtocol` | 设备 | 2 | 1 | N8 | C |  | UiRun.kt:40,128；TreeCollector.kt:348；UiPostAction.kt:180 |  | 采集协议异常；设备端多处按它分支 |
| `connectionBusy` | 主机 | 2 | 2 | N4 | C | SKILL.md:24；connection.md:93 |  |  | 主机连接被别的会话的请求占着 |
| `connectionClosed` | 主机 | 1 | 2 | N4 | C |  |  |  | 发出前连接已关，未发出 |
| `connectionFailed` | 主机 | 0（+1） | 2 | N6 | C |  |  |  | 连不上设备：检查网络、Gateway |
| `covered` | 设备 | 1 | 2 | N2 | C | actions.md:39 |  |  | 目标不在操作窗口，先 snapshot 看范围 |
| `credential` | 设备 | 1 | - | N9 | I |  |  |  | 推断：配对注册协议校验 |
| `daemonCleanup` | 主机 | 1 | - | N9 | D |  |  |  | 守护进程状态文件清理 |
| `daemonStopped` | 主机 | 1 | 2 | N5 | C |  | app.py:918,939 |  | 主机对 daemon status/stop 按它特殊处理 |
| `daemonUnavailable` | 主机 | 1 | 2 | N5 | C |  |  |  | 守护进程没有给出可用端点 |
| `daemonVersion` | 主机 | 1 | 2 | N5 | C |  |  |  | 停掉旧版本守护进程 |
| `deviceCleanup` | 主机 | 1 | - | N9 | D |  |  |  | 推断：文件任务设备端清理未确认 |
| `deviceName` | 主机 | 1 | 2 | N1 | C |  |  |  | 名字已被占用，换 --name；候选并入 arguments |
| `deviceNotFound` | 主机 | 3 | 2 | N5 | C |  | coordinator.py:298 |  | devices 后指定 --device；主机内部按它分支 |
| `deviceRequired` | 主机 | 2 | 2 | N5 | C |  |  |  | 混合：没有配对设备（要配对）／多台未选（要选） |
| `displayMismatch` | 设备 | 1 | 2 | N5 | C |  |  |  | 先 session close 或 --replace |
| `displayUnavailable` | 设备 | 3 | 2 | N8 | C |  |  |  | 主屏或新建后台屏不在 ON |
| `displayUnsupported` | 设备 | 6 | 2 | N3 | C |  |  |  | 混合：平台或权限不支持后台屏，改用主屏 |
| `duration` | 设备 | 1 | 2 | N1 | C |  |  |  | 缩短手势或加大 --timeout；候选并入 arguments |
| `explicitRowClick` | 设备 | 1 | 2 | N3 | C |  |  |  | 改点所在行并核对勾选 |
| `externalTasks` | 设备 | 1 | 2 | N1 | C |  |  |  | 须按用户意图选 --return-tasks 或 --discard-tasks |
| `fileCleanup` | 主机 | 2 | 1 | N9 | D |  |  |  | 混合：临时文件清理未确认／目标身份变了 |
| `fileExists` | 两端 | 3 | 2 | N1 | C |  |  |  | 显式 --replace 或换路径 |
| `fileParent` | 主机 | 2 | 2 | N5 | C |  |  |  | 建目录或换路径 |
| `fileType` | 主机 | 2 | 2 | N1 | C |  |  |  | 换成普通文件路径 |
| `gatewayNotReady` | 设备 | 1 | - | N9 | U |  |  |  | 管理界面打开配对窗口时 |
| `gatewayStopping` | 设备 | 5 | 2 | N6 | U |  |  |  | 多数在管理界面；转发时一处给调用者 |
| `hash` | 两端 | 2 | 1 | N8 | C |  |  |  | 传输完整性 |
| `helperSlots` | 设备 | 2 | 2 | N8 | C |  |  |  | shell 辅助进程容量锁不可用 |
| `history` | 主机 | 4 | 1 | N5 | C |  |  |  | 混合：缓存目录不可用／历史忙／时间不够 |
| `identityChanged` | 主机 | 1 | 2 | N6 | C |  |  |  | 配对时证书与已存不同；候选并入 tlsIdentity |
| `identityUnknown` | 设备 | 12 | 2 | N2 | C |  |  |  | 身份事实读不到，重新 snapshot（可能持续） |
| `imageFormat` | 主机 | 1 | 1 | N8 | C |  |  |  | 设备给出的图像类型不符 |
| `imagePath` | 主机 | 1 | 2 | N5 | C |  |  |  | 图像输出目录不存在；候选并入 fileParent |
| `imageUnavailable` | 设备 | 5 | 2 | N4 | C |  |  |  | 后台屏截图失败；候选并入 screenshotUnavailable |
| `inputFocus` | 设备 | 6 | 2 | N3 | C |  |  |  | 目标不是当前输入焦点 |
| `inputUnavailable` | 设备 | 5 | 2 | N5 | C |  |  |  | 混合：屏幕关闭或锁定（wake／请用户解锁）／输入服务不可用 |
| `internal` | 主机 | 0（+1） | 3 | N8 | C |  |  |  | 主机文件任务未预料的异常 |
| `internalError` | 主机 | 0（+1） | 3 | N8 | C |  |  |  | 主机本地服务未预料的异常 |
| `invalidRequest` | 主机 | 2 | 2 | N8 | I |  |  |  | 本地服务收到格式不对的请求 |
| `jobOwner` | 两端 | 2 | 2 | N5 | C |  |  |  | 用发起它的会话或设备查询 |
| `jobUnavailable` | 两端 | 2 | 2 | N8 | C |  |  |  | 任务不在保留表里（编号错或已丢） |
| `jsonLimit` | 两端 | 3 | 2/3 | N1 | C |  |  |  | 请求或回执超出 JSON 上限 |
| `knownDevice` | 主机 | 1 | 2 | N5 | C |  |  |  | 推断：已知设备改用保存的身份 |
| `localConfiguration` | 主机 | 2（+1） | 2 | N6 | C |  |  |  | 主机本地配置读写失败 |
| `magnificationActive` | 设备 | 2 | 2 | N6 | C |  |  |  | 放大功能开着，坐标输入被拒 |
| `magnificationOverlay` | 设备 | 1 | 2 | N6 | C |  |  |  | 放大覆盖层还在；候选并入 magnificationActive |
| `mappingUnknown` | 设备 | 10 | 2 | N3 | C |  |  |  | 混合：坐标映射事实缺失（改用引用）／非 user 0 |
| `messageLimit` | 两端 | 2 | 2/3 | N1 | C |  |  |  | 请求或回执超出网络消息上限；候选并入 jsonLimit |
| `method` | 两端 | 2 | 2 | N8 | C |  |  |  | 版本不一致导致的未知方法 |
| `nodeLimit` | 设备 | 2（+2） | 2 | N2 | C | actions.md:30 | UiObservation.kt:63；UiSelector.kt:103 |  | 也是观察缺口词汇；缩小范围 |
| `noPublishedSnapshot` | 设备 | 2 | 2 | N2 | C |  | UiCondition.kt:335 |  | 没有已发布的观察；候选并入 noSnapshot |
| `noSnapshot` | 设备 | 2 | 2 | N2 | C |  |  |  | 没有当前观察，先 snapshot |
| `notificationsUnavailable` | 设备 | 1 | 2 | N8 | C |  |  |  | 通知服务不可用 |
| `notReady` | 设备 | 10 | 2 | N3 | C |  |  | 7 | 混合：滚到可见／换列出的方向／等待 |
| `offset` | 两端 | 3 | 1 | N8 | C |  |  |  | 传输完整性 |
| `output` | 主机 | 6（+2） | 1 | N8 | C |  |  |  | 混合：输出完整性／取输出时间不够 |
| `outputExpired` | 设备 | 1 | 2 | N8 | C |  |  |  | 输出已不保留，命令不重复执行 |
| `pairingExpired` | 设备 | 1 | 2 | N6 | C |  |  |  | 配对窗口已关闭，请用户显示新配对码 |
| `pairingInvalid` | 主机 | 2 | 2 | N5 | C |  |  |  | 保存的配对记录坏了：devices、forget 后重配 |
| `pairingOnly` | 设备 | 1 | - | N9 | I |  |  |  | 推断：配对专用连接上发了别的请求 |
| `pairingStore` | 设备 | 1 | - | N9 | U |  |  |  | 管理界面：配对存储失败 |
| `pairingUnconfirmed` | 主机 | 2 | 2 | N5 | C |  |  |  | 用当前配对码再 pair 一次 |
| `partialBaseline` | 设备 | 1 | 2 | N2 | C |  | UiCondition.kt:335 |  | screen-change 基线必须完整 |
| `point` | 设备 | 2 | 2 | N1 | C |  |  |  | 混合：坐标在图像外（改参数）／引用中心在屏外（滚到可见） |
| `privatePermissions` | 主机 | 5 | 2 | N6 | C |  | private_files.py:227 |  | 主机私有目录权限不对 |
| `privatePermissionsRepair` | 主机 | 1 | 2 | N6 | C |  |  |  | 收紧权限失败 |
| `protected` | 设备 | 4 | 2 | N3 | C |  |  |  | 不能操作 AndroidCtl 自己的界面 |
| `protocol` | 主机 | 5 | 2/3 | N8 | C |  |  |  | 线上协议不符 |
| `protocolMismatch` | 两端 | 2（+1） | 2 | N6 | C | connection.md:101 | transport.py:270 |  | 主机和 App 版本不配套 |
| `publishFailed` | 设备 | 1 | 1 | N8 | C |  |  |  | 发布失败（系统错误号）；仅 EIO 时结果未知（3） |
| `publishUnsupported` | 两端 | 4 | 1 | N1 | C | jobs-files.md:63 | file_jobs.py:417 |  | 两端一致：换 --replace 或换路径；主机按它分支 |
| `query` | 主机 | 0（+1） | 1 | N4 | C |  |  |  | 任务查询失败，再查一次 |
| `queueFull` | 设备 | 1 | 2 | N4 | C |  |  |  | 网关队列满，未派发 |
| `radioUncheck` | 设备 | 1 | 2 | N3 | C |  |  |  | 单选不能直接取消，改选别项 |
| `range` | 设备 | 3 | 2 | N1 | C |  |  |  | 混合：值越界（改参数）／最小最大值读不到（不能用 range） |
| `rangeUnavailable` | 设备 | 1 | 2 | N3 | C |  |  |  | 目标没有范围信息 |
| `region` | 设备 | 1 | 2 | N1 | C |  |  |  | 混合：坐标区域越界（改参数）／引用区域超出屏幕（滚到可见） |
| `replyUnavailable` | 主机 | 1 | 3 | N7 | C |  |  |  | 请求可能已到达，不得重发 |
| `requestEnded` | 设备 | 1 | - | N9 | I |  | TargetManagement.kt:164 |  | 请求已返回后的内部检查 |
| `revoked` | 设备 | 2 | 2 | N6 | C |  |  |  | 配对已被撤销 |
| `revokeUnconfirmed` | 设备 | 1 | - | N9 | U |  |  |  | 管理界面撤销配对后等待确认 |
| `rootUnavailable` | 设备 | 2（+1） | 2 | N4 | C |  | UiPostAction.kt:121 |  | 窗口根节点读不到，也是观察问题码 |
| `routeUnavailable` | 设备 | 2 | 1 | N3 | C |  |  | 1 | scroll-until 中途某方向已无路可走 |
| `scopeMismatch` | 设备 | 3 | 2 | N1 | C |  | UiCondition.kt:335 |  | 按提示补 --window／范围参数 |
| `scopeUnknown` | 设备 | 3 | 2 | N1 | C |  | UiCondition.kt:458；UiScope.kt:56；UiActions.kt:931；UiSignals.kt:17；UiBackend.kt:385 | 8 | 按提示 snapshot --window N；也常作成功回执的诊断 |
| `screenshotUnavailable` | 设备 | 4 | 2 | N4 | C |  |  |  | 主屏截图失败 |
| `scriptFile` | 主机 | 1 | 2 | N1 | C |  |  |  | 脚本文件读不到；候选并入 arguments |
| `selectionInvalid` | 主机 | 1 | 2 | N5 | C |  |  |  | device select 重设选择 |
| `sessionChanged` | 设备 | 6 | 2 | N5 | C |  | FileJob.kt:249 |  | 混合：先看 status／管理界面刷新／会话已被关闭 |
| `sessionReleased` | 设备 | 2 | 2 | N5 | C |  |  |  | 设备上有人释放了会话，明确重开 |
| `sessionRequired` | 设备 | 1 | 2 | N5 | C | displays.md:20 |  |  | 没有或已到期，明确 session open |
| `shizukuMode` | 设备 | 1 | 2 | N6 | U |  |  |  | 多数在管理界面：root 模式或身份不明 |
| `shizukuPermission` | 设备 | 2 | 2 | N6 | C |  |  |  | 请用户在 Shizuku 里授权 |
| `shizukuUnavailable` | 设备 | 2 | 2 | N6 | C |  |  |  | 请用户启动 Shizuku |
| `size` | 设备 | 3 | 1 | N8 | C |  |  |  | 传输完整性 |
| `snapshotBudget` | 设备 | 8 | 2/1 | N4 | C |  | UiCondition.kt:458 |  | 采集或条件计算时间用尽 |
| `snapshotPath` | 主机 | 1 | 2 | N5 | C |  |  |  | 观察输出目录不存在；候选并入 fileParent |
| `sourceChanged` | 两端 | 6 | 1 | N4 | C | jobs-files.md:61 |  |  | 传输中源文件变了，源稳定后重传 |
| `sourceUnknown` | 设备 | 4 | 2 | N2 | C |  |  |  | 混合：重新截图或 snapshot／App 按键需要输入焦点窗口 |
| `stale` | 两端 | 13 | 2 | N2 | C | images.md:108；reference.md:84 | UiRefRepair.kt:50（subject=ref） | 3 | subject=ref/image；subject：image/ref |
| `staleWorker` | 设备 | 1 | - | N9 | U |  |  |  | 管理界面强制退出前执行器已换 |
| `startupFailure` | 主机 | 1 | 2 | N8 | C |  |  |  | 守护进程启动失败 |
| `stateUnknown` | 设备 | 1 | 2 | N8 | C |  |  |  | 显示状态读不到 |
| `stopping` | 设备 | 2 | 2 | N6 | C |  |  |  | 执行器正在停止 |
| `submitUnsupported` | 设备 | 2 | 2 | N3 | C |  |  |  | 不能提交，改用按键或按钮 |
| `taskOnOtherDisplay` | 设备 | 1 | 2 | N6 | C |  |  |  | 消息要求先问用户 |
| `taskPreflightUnknown` | 设备 | 1 | 2 | N3 | C |  |  |  | 缺权限，不能用后台屏任务检查 |
| `taskReturn` | 设备 | 3 | 1 | N8 | C |  |  |  | 送回主屏失败，告诉用户 |
| `temporaryChanged` | 两端 | 2 | 1 | N4 | C |  |  |  | 临时文件身份变了，重传 |
| `temporaryCleanup` | 主机 | 2 | 1 | N9 | D |  | file_jobs.py:542 |  | 主机临时文件清理未确认；主机按它分支 |
| `timeout` | 两端 | 18 | 2/3 | N4 | C |  | TargetManagement.kt:164；ClipboardOperations.kt:87；UiCondition.kt:458 |  | 混合：时间用尽／连不上／设备端超时取值越界（参数） |
| `tlsIdentity` | 主机 | 3（+1） | 2 | N6 | C |  |  |  | 混合：证书变了（核实后 device trust）／时钟问题 |
| `transfer` | 主机 | 1 | 1 | N8 | C |  |  |  | 通用传输失败，读消息 |
| `transferState` | 设备 | 1 | 2 | N9 | I |  |  |  | 推断：传输步骤顺序不对 |
| `treeByteLimit` | 设备 | 3（+2） | 2 | N2 | C | actions.md:30 | UiObservation.kt:63；UiSelector.kt:103 |  | 也是观察缺口词汇；缩小范围 |
| `uiCleanup` | 设备 | 5（+1） | 2 | N6 | C | displays.md:95（作为诊断字段） |  |  | 混合：界面清理未完成挡住新工作；也是诊断字段 |
| `unauthorized` | 两端 | 0（+2） | 2 | N6 | C |  | coordinator.py:430 |  | 混合：配对窗口被拒（新配对码）／凭据被拒；主机按它分支 |
| `unexpectedTaskMove` | 设备 | 1 | 3 | N7 | C | actions.md:180；displays.md:53 | ui_output.py:232 |  | 启动已发生，不得重复；主机按它换提示 |
| `userScope` | 设备 | 5 | 2 | N6 | C |  |  |  | 须在 user 0 下 |
| `valueAlreadyMatches` | 设备 | 1 | 2 | N0 | C |  |  |  | 勾选状态已满足，不要点行 |
| `windowsUnavailable` | 设备 | 1 | 2 | N4 | C |  |  |  | 窗口枚举失败 |
| `windowUnavailable` | 设备 | 0（+1） | - | N2 | D |  |  |  | 请求的窗口不在目录里（观察问题码） |
| `workerBusy` | 设备 | 1 | - | N9 | U |  |  |  | 推断：执行器锁被占，管理界面显示 |
| `workerDirectory` | 设备 | 1 | - | N9 | U |  |  |  | 推断：执行器目录创建失败 |
| `workerExitUnconfirmed` | 设备 | 1 | - | N9 | U |  |  |  | 管理界面强制退出未确认 |
| `workerFailure` | 设备 | 0（+1） | 3 | N8 | C |  |  |  | 设备端未预料的异常（默认码） |
| `workerLockChanged` | 设备 | 1 | 2 | N6 | C |  |  |  | 执行器锁变化，拒绝新工作 |
| `workerUnavailable` | 设备 | 4 | 2 | N6 | C |  |  |  | 请用户在设备上 Refresh status |
| `workspace` | 主机 | 7 | 2 | N5 | C |  |  |  | 混合：工作区目录不存在／文件读不了 |

类别计数：N0 1 个；N1 17 个；N2 13 个；N3 17 个；N4 15 个；N5 20 个；N6 26 个；N7 3 个；N8 27 个；N9 17 个
