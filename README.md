1.开发环境
Unity版本：Unity 2022.3.62f1c1
脚本使用C#编写，本demo未使用第三方代码或资产
2.核心脚本
①PlayerController：包含基于摄像机方向的移动、奔跑、二段跳、滑铲及贴墙滑行等跑酷动作。
②StartMenu：脚本管理主菜单，包含开始游戏（转场提示按任意键进入）、设置调节（音量、灵敏度、分辨率）及退出功能。
③PatrolAI：控制机器人巡逻：在A、B点间移动，当玩家进入触发范围时显示警告文本，离开后恢复正常巡逻。
④NPC_RingQuest：实现戒指支线任务：NPC对话（按F），首次交互接受任务并生成戒指，玩家拾取戒指后再次交互完成任务，更新对话状态。
⑤VideoFullscreenTrigger：用于触发过场动画：玩家交互后播放视频，结束后激活后续交互物体。
⑥RopeTrigger：用于训练场终点滑索：显示鼓励文本、播放视频并传送玩家至高塔，完成支线到主线的衔接。
⑦MovingPlatform：实现平台在两个点之间往返移动，用于跑酷路线中的动态障碍或辅助跳跃。
3.核心技术实现细节
3.1相机跟随优化
视角限制、平滑插值及疯狂转圈的解决方案，相机跟随优化，其主要细节如下：
①视角限制：yRotate = Mathf.Clamp(yRotate, -15, 90) 限制俯仰角，防止相机翻转。
②平滑跟随：Vector3.Lerp(transform.position, targetPos, smoothSpeed * Time.deltaTime)。
③Y轴反转：yRotate += -Input.GetAxis("Mouse Y") * rotateSpeed，强制鼠标上下移动时视角方向符合直觉。
3.2玩家跑酷物理与碰撞处理
角色移动的物理参数调校、碰撞体分层、蹬墙跳和滑铲的状态切换逻辑。角色移动基于 CharacterController 组件，通过手动模拟重力和速度实现跑酷动作，其主要细节如下：
①物理参数调校：moveSpeed / runSpeed 控制地面移动速度；jumpHeight 转换为初速度（Mathf.Sqrt(jumpHeight * -2f * gravity)）；gravity 常量每帧累加到 velocity.y，实现类真实下落曲线。贴墙滑行时单独使用 currentWallSlideSpeed 替代重力，并设置加速度与最大速度上限。
②碰撞体分层：通过 LayerMask 的 wallLayer 区分可交互墙体（用于贴墙滑行和蹬墙跳）。CharacterController 自带的 isGrounded 用于地面检测，SphereCast 四方向（前、后、左、右）检测墙面。
③蹬墙跳状态切换逻辑：CheckWall() 方法使用 Physics.SphereCast 向四个方向发射球形射线，命中 wallLayer 则 isWallSliding = true。空中且贴墙时，velocity.y 被替换为 -currentWallSlideSpeed（减速滑落），同时重置 jumpCount = 1，允许玩家从墙上反跳。跳跃时通过 jumpCount > 0 条件实现二段跳与蹬墙跳的共用计数。
④滑铲状态切换逻辑：滑铲触发条件：地面 + 按 LeftShift + 按 C + 有移动输入。触发后 slideCounter 计时，期间移动方向锁定为 slideDir，速度固定为 slideSpeed，模型 Y 轴缩放用 Lerp 平滑过渡到 slideScaleY，模拟下蹲滑行。滑铲结束或中断后恢复原比例和正常移动。
⑤地面与空中状态机：isGrounded 为真时重置 jumpCount、currentWallSlideSpeed，并强制 velocity.y = -2f 保证贴地。isWallSliding 与 isGrounded 互斥，优先处理贴墙逻辑。
3.3对于UI交互与事件绑定
设置面板实时修改：Slider.onValueChanged.AddListener绑定SetVolume、SetSensitivity,Dropdown.onValueChanged绑定SetResolution。音量直接修改 AudioListener.volume；灵敏度存入静态变量 mouseSensitivity 供 CameraFollow 使用；分辨率调用 Screen.SetResolution。
ESC 菜单：游戏中按 ESC 时，直接激活 StartMenu 中的设置面板（同一份 UI），不涉及额外的暂停逻辑或射线穿透处理。
开始游戏转场：点击开始按钮后显示转场黑屏（transitionPanel），提示“按任意键进入”，等待 Input.anyKeyDown 后加载游戏场景。
3.4触发式剧情系统
区域触发器：所有交互点使用 OnTriggerEnter / OnTriggerExit 检测 Player 标签，显示/隐藏“按F”提示。
过场动画播放：使用 VideoPlayer，交互时调用 Play()。VideoFullscreenTrigger 在 Update 中检测 !isPlaying && frame > 0 判断播放结束；RopeTrigger 使用协程等待视频时长后传送。
倒计时机制（主线跑酷）：塔底触发剧情后启动倒计时（设计中 120 秒，超时触发假结局并送回出生点）。
多结局分支：BossDialogueAndChoice 中，玩家按 F 逐句显示 BOSS 台词，结束后激活左右压力板。压力板分别调用 TriggerGoodEnding() 或 TriggerBadEnding()：好结局显示成功文本；坏结局显示失败文本并传送玩家回出生点。
4.AI辅助内容
本Demo在快速开发、质量表现上使用AI辅助，不影响核心逻辑原创性：
AI辅助过场动画生成使用AI生成塔顶BOSS战斗、机器人守卫、角色过场动画，提升游戏视觉表现。主要用于：
①BOSS登场动画;②机器人守卫转移动画;③结局演出画面;④绳索转移画面；⑤假下水道动画等
另外用于：
优化代码结构，修复BUG（如UI点不动、相机旋转、灵敏度不生效，视频重复播放等），提供简洁稳定的实现方案，关卡象征意义、NPC对话、BOSS交涉台词通过AI辅助完善，保证叙事完整统一。
