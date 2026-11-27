# 图形界面操作（Linux / AT-SPI）
STATE ✓ 2026-11-27 实测 Bazzite Fedora45 · KDE Wayland · flatpak Qt6｜SCOPE Linux 桌面与 flatpak 应用的控件树与动作｜SCRIPT `lin_ui.py`
MARK 核 = 冲突时先保；无标 = 一般

## 何时用
| 场景 | 处置 |
|---|---|
| Linux 上点 GUI / flatpak 应用自动化 | 先 `prep`，再 `tree` 定点 |
| Wayland 下读控件、窗口坐标对不上、`xdotool` 无效 | 走 AT-SPI + `act` |
| 只能看截图、量不准位置 | 用 `act` 绕开坐标 |

## 命令
```bash
cd ~/.config/cli-agent/skill
./lin_ui.py prep                          # 体检：toolkit-accessibility / a11y bus / 会话 / ydotool / backend
./lin_ui.py tree --app konsole --max 200  # 枚举控件 TSV：id role name extents actions flags
./lin_ui.py act --id 6132db69             # Wayland 首选：走 Action，不动鼠标
./lin_ui.py act --id af4b9ead --name Press
./lin_ui.py click --id 6132db69           # 坐标点击；Wayland 需 --force，且要 ydotoold 在跑
./lin_ui.py shot -o s.png                 # grim → spectacle → gnome-screenshot → import 自适应
flatpak override --user --talk-name=org.a11y.Bus <appid>   # 沙箱应用上不了树时才补
```

## 参数 / 退出码
| 参数 | 说明 |
|---|---|
| `--app` / `--max` / `--depth` | 限定应用 / 节点上限（默认 300）/ 递归深度（默认 25） |
| `--id` / `--name` | `tree` 输出的 8 位 id / `act` 的 action 名，缺省取第 0 个 |
| `--force` | 坐标疑似局部、或在 Wayland 硬走 xdotool 时放行 |
| 码 | `0` 成功 ｜ `2` 无 a11y 绑定或总线 ｜ `4` 无命中 ｜ `5` 能力/参数不支持 ｜ `6` id 或 action 未命中 |

## 失败处置
| 现象 | 原因 | 动作 |
|---|---|---|
| `prep` 报 no bus | `at-spi-bus-launcher` 未起 | `gsettings set org.gnome.desktop.interface toolkit-accessibility true`，重登 |
| 树里只有 `application` 节点或 exit 4 | 应用没接 a11y 总线 | Qt 应用导出 `QT_LINUX_ACCESSIBILITY_ALWAYS_ON=1` 重启；flatpak 补 talk-name |
| `flags=local` | Wayland 给的是窗口局部坐标 | 改用 `act`，别 `click` |
| `click` 报 xdotool 空操作 | Wayland 下 xdotool 动不了指针 | 起 `ydotoold` 走 ydotool，或改用 `act` |
| `click` 报无注入器 | 两者都没装 | ostree：`rpm-ostree install ydotool`；Debian：`apt install ydotool` |

## 陷阱
- `get_extents(SCREEN)` 与 `get_extents(WINDOW)` 返回同一个窗口局部值，只有 `PARENT` 是父相对；`local` 标志只认 `(0,0)`，窗口不在原点时漏判，`click` 会静默点偏。
- 截图分辨率可高于屏幕逻辑分辨率（缩放），AT-SPI 坐标与截图像素不可换算混用。
- 枚举依赖应用自己实现 a11y：flatpak Qt6（KCalc）有完整树；konsole 不带 `QT_LINUX_ACCESSIBILITY_ALWAYS_ON=1` 完全不上总线；Electron 需开关；自绘界面（游戏）没树。
- `act` 连发要留间隔：0.6 s 内连按 3 次只进 1 个事件，循环里加 `sleep 0.5` 再读回校验。
- `--depth` 决定召回：kcalc 按钮在第 8 层，`--depth 3` 只剩 11 个节点。
- 被枚举的应用先杀掉再收工：残留窗口会污染后续 `tree`。
