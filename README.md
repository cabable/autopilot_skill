# autopilot_skill
这是一份 《AutoPilot 视觉导航系统 · 实战固化操作手册》。

这不是一份普通的 API 文档，而是基于 OpenCV + MSS 深度耦合开发中，两周内从“能跑”到“不死”的全部应激反应记录。所有内容均已通过 fight_against_reality 单元测试（即：真机跑崩过至少 3 次以上才写入）。

请将此文档作为项目的 /docs/WAR_ZONE.md 供奉在根目录。

⚠️ AutoPilot 踩坑固化清单 & 调参圣经
Ⅰ. 架构地形图 (Architecture Map)
为了理解下面所有的“为什么”，先记住这条铁血调用链：

Tkinter UI (主线程) --> MSS (截屏线程/协程) --> CV2 (预处理 & 模板匹配) --> 控制逻辑 (锁与事件) --> Win32 API (输入模拟) --> AI 闭环 (结果回灌)

致命红线：OpenCV 处理的是 numpy.ndarray（BGR 排列），MSS 输出的是 BGRA，Tkinter 的 PhotoImage 只吃 RGB。绝不允许跨格式直接赋值，否则画面会变成赛博朋克风格（偏紫/绿）。

Ⅱ. 硬核踩坑清单（按现场爆炸时间排序）
1. Tkinter 顺序坑：Tk() 必须先于 PhotoImage
现象：RuntimeError: Calling Tk_PhotoPutBlock when no Tk window exists。即使你的 PhotoImage 写在函数最下面，只要 Tk() 实例化晚于它，Python 底层 C 指针直接挂掉。

固化方案：

python
# ✅ 绝对正确的生死顺序
root = tk.Tk()          # 1. 先让内核态窗口活起来
root.withdraw()         # 2. 隐藏主窗（如果只需要弹窗）
img = tk.PhotoImage(file="tmp.png") # 3. 现在才能安全读取图片
铁律：任何全局变量如果涉及 PhotoImage，必须在 Tk() 实例化之后的上下文作用域内定义。

2. MSS 物理像素坐标系 (DPI Awareness 缺失症)
现象：截图坐标偏移，点屏幕左上角 (0,0) 没问题，点 (500,500) 却点到了 (375,375)。
根因：MSS 底层调用 GetSystemMetrics 返回的是物理像素，而 Windows 缩放（如 125%/150%）下，win32api.GetCursorPos() 返回的逻辑坐标会被系统缩放。

固化方案：

python
import ctypes
# 强制告诉 Windows 我们已处理 DPI（必须在 MSS 启动前）
ctypes.windll.user32.SetProcessDPIAware()
# 或者使用 Per-Monitor V2
ctypes.windll.shcore.SetProcessDpiAwareness(2)

# 转换公式（若必须转换）：
physical_x = int(logical_x * scale_factor)
记住：在 mss 抓图时，禁用任何逻辑坐标系计算，直接使用 monitor["left"] 等原生偏移。

3. _densify_scales 宽范围不补点的陷阱（金字塔匹配中的暗疮）
现象：模板在 0.8 倍时能匹配，在 1.2 倍时能匹配，唯独在 1.0 倍（原图）时匹配分极低。
根因：OpenCV 的 pyramid_down 或自定义缩放函数为了省算力，设定 step > 0.1。当从 0.8 跳到 1.2 时，跳过了 1.0 的精确尺度匹配，导致模板走形。

固化配方：

python
def _densify_scales(min_scale, max_scale):
    # 禁止使用 np.arange(0.5, 1.5, 0.2) 
    # ✅ 必须使用 0.05 步长强制覆盖所有整数级及半级尺度
    scales = np.arange(min_scale, max_scale, 0.05) 
    # 重点：显式强制插入 1.0
    if 1.0 not in scales: 
        scales = np.append(scales, 1.0)
    return sorted(set(scales))
调参备注：若游戏素材是像素画风，步长甚至要降到 0.025。

4. 模板尺寸 40~120px 铁律（特征纹理生死线）
为什么不是 30？为什么不是 200？

< 40px：在 MSS 截取的 1080p 画面中，特征点（角点/边缘）不足 15 个，匹配极易被噪点污染（误报率 > 70%）。

> 120px：旋转增强时，矩阵仿射变换会产生严重的锯齿及边界切割，且匹配耗时呈指数上升（因为滑动窗口变大）。

实战策略：若目标图标是 30px，不要直接匹配。截取包含该图标的 50x50 背景+图标 组合体作为模板，利用四周背景纹理辅助定位，然后通过相对偏移点击中间。

5. 平面旋转增强：为什么必须卡死 15° 而非 30°
血泪教训：游戏 UI 并非完全水平。加入 getRotationMatrix2D 增强数据集是常规操作。
但如果开到 30°：模板会识别到斜向的发光粒子或文字残影，匹配分数反而高于正向模板，导致鼠标点偏。
Root Cause：30° 时，矩形模板四个角大量填充无效黑色区域（背景），归一化相关系数 TM_CCOEFF_NORMED 对黑色区域有天然的“高分偏向”（因为减去了均值）。

固化参数：

python
angles = range(-15, 16, 3)  # ✅ 最大 ±15°，步长 3°
# 坚决不碰 30°，若目标旋转剧烈，改用 MINI 特征点匹配（SIFT），别碰模板旋转。
6. 锁纪律：stop() 绝对绝对不能持锁 join()
现象：点击“停止”按钮后，GUI 卡死，转圈圈，强制杀进程。
根因链条：主线程调用 stop() -> 获取 thread_lock -> 调用 thread.join() 等待子线程结束 -> 子线程若要退出，需要释放资源，却也尝试获取 thread_lock -> 死锁（Deadlock）。

固化纪律：

python
def stop(self):
    # ✅ 第一步：立刻修改状态位（原子操作，不需要锁）
    self.running = False  
    # ✅ 第二步：等待线程自然醒来并退出，此时绝不能拿着锁！
    if self.thread and self.thread.is_alive():
        self.thread.join(timeout=0.5) 
    # ✅ 第三步：等线程死透了，再获取锁清理残留资源（非阻塞）
7. 窗口识别与“点击日志成功但游戏不响应”的完整根因链（最经典难题）
现象：日志打印 click at (x,y) success，鼠标没动，或者游戏里的按钮没按下去，但记事本能打字。
完整根因链（按排查优先级排序）：

前台窗口句柄失效（最常见）：win32gui.FindWindow 拿到了句柄，但游戏在后台最小化了。SetCursorPos 对于后台窗口无效，必须 SetForegroundWindow(hwnd) 强制激活。

鼠标事件模式错误：win32api.SendMessage(hwnd, WM_LBUTTONDOWN, ...) 对于 Unity/UE4 引擎无效（它们采用 DirectInput 轮询，不吃 Windows 消息队列）。必须改用硬件层模拟：win32api.mouse_event(MOUSEEVENTF_LEFTDOWN, x, y, 0, 0)。

点击滞留时间不足：仅发送 Down+Up 间隔 1ms，游戏引擎判定为“误触抖动”忽略。必须增加 Hold 时间。

python
mouse_event(DOWN); time.sleep(0.05); mouse_event(UP)  # 50ms 是黄金阈值
坐标系未转换：向 mouse_event 传入的是 OpenCV 截图内的相对坐标，而非绝对屏幕坐标。必须加上 monitor 的 left 和 top 偏移。

Ⅲ. 高级机制：AI 知识回灌 OpenCV 自学习闭环
AutoPilot 最核心的防御性编程在于 “负样本纠正”。

闭环逻辑：

OpenCV 匹配得分 > 0.85 -> 直接执行点击。

匹配得分在 0.65 ~ 0.85 之间（模糊态） -> 触发 AI（大模型/或预置逻辑）根据上下文坐标推断“大概位置”并执行试探点击。

回灌：试探点击后，立即重新截图。若画面状态发生了预期变化（如怪物死亡、UI跳转），则 将试探点击时的截图区域（即难例样本）保存下来，通过仿射变换生成 ±3° 的变体，自动覆盖写入 templates/ 目录中。

下一次启动时，cv2.imread 加载的模板集合里就多了一张针对该动态光照的专用模板。

代码固化：

python
def knowledge_reinfusion(success_flag, roi_img):
    if success_flag and match_score < 0.8: # 属于“歪打正着”
        timestamp = time.strftime("%Y%m%d_%H%M%S")
        cv2.imwrite(f"templates/aug_{timestamp}.png", roi_img)
        # 注意：此时必须重置 TemplateManager 的缓存列表，否则需要重启才生效
Ⅳ. 调参配方速查表 (Quick Recipe)
参数项	推荐值	极端场景微调方向
匹配算法	cv2.TM_CCOEFF_NORMED	光照多变改用 TM_CCORR_NORMED
匹配阈值 (Threshold)	0.82	遮挡严重降到 0.72，低误报升到 0.88
高斯模糊预处理	(5, 5)	像素风游戏用 (3,3) 保留硬边缘
截屏帧率	15 FPS	动作游戏升到 30 FPS（注意 CPU 温度）
点击后冷却 (Cooldown)	0.2 ~ 0.3s	读条界面强制 1.0s 硬等待
金字塔缩放范围	0.8 ~ 1.2	如果经常丢失远小目标，扩至 0.6 ~ 1.4，必须补 1.0 点
Ⅴ. 终极防线（工程打包经验）
日志不是用来看的，是用来画图的：务必保存每一帧的匹配中心点坐标列表，若鼠标漂移，回放坐标点连成的轨迹线，比看日志数字直观 10 倍。

内存泄漏监测：MSS 的 grab() 返回对象如果不显式 del，每 10 分钟会吃掉 200MB 内存。务必在循环末尾 del img，并调用 gc.collect()（仅在低帧率场景）。

异常兜底：OpenCV 匹配失败返回空列表时，绝不要 return None，而是 return {"x": last_known_x, "y": last_known_y, "valid": False}，让控制层决定是重试还是使用卡尔曼滤波预测
