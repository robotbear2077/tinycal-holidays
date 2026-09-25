# tinycal-holidays

给 macOS 菜单栏日历 **[TinyCal（小历）](https://apps.apple.com/cn/app/id1114272557)** 补上最新的中国法定节假日和调休数据。

TinyCal 很久没更新了，内置的节假日数据只到 **2022 年**。从 2023 年起，日历上既不显示「休」，也不显示「班」。这个脚本不修改 app 本体，只改写 TinyCal 自己的缓存文件，把最新的放假安排补进去。

```
$ tinycal-holidays
2025: 已获取 33 条节假日/调休
2026: 已获取 39 条节假日/调休
2027: 官方尚未发布
已备份原缓存到 ~/Library/Containers/app.cyan.tinycalx/Data/Documents/calendars.bak
共 13 个月份缓存，修改了 2 个。
已重启 TinyCal。
```

## 特点

- **不改 app 本体**：不动二进制，不重新签名，也不注入代码。App Store 版可以直接用，更新 TinyCal 也不受影响。
- **数据跟着官方更新**：数据来自 [NateScarlet/holiday-cn](https://github.com/NateScarlet/holiday-cn)。这个项目会根据国务院办公厅发布的放假通知自动更新。
- **没有依赖**：一个 Python 3 脚本，只用标准库，macOS 自带的 `python3` 就能运行。
- **可以重复运行，也可以还原**：多次运行结果一样。第一次运行时会自动备份原缓存，用 `--restore` 可以一键还原。

## 安装

```bash
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/robotbear2077/tinycal-holidays/main/tinycal-holidays \
  -o ~/.local/bin/tinycal-holidays
chmod +x ~/.local/bin/tinycal-holidays
```

如果 `~/.local/bin` 不在 `PATH` 里，可以用完整路径 `~/.local/bin/tinycal-holidays` 运行，也可以把它加进 `PATH`。

## 使用

1. 打开 TinyCal，**把想看的月份都翻一遍**。TinyCal 只会为浏览过的月份生成缓存，一般会预先缓存当前月前后一年左右。
2. 运行：

   ```bash
   tinycal-holidays                # 补丁，然后自动重启 TinyCal
   tinycal-holidays --no-restart   # 只补丁，不重启
   tinycal-holidays --restore      # 还原成原始缓存
   ```

### 什么时候需要再运行一次

- 国务院发布了新一年的放假安排，一般在每年 11 月前后
- 翻到了之前没看过的月份
- 在 TinyCal 设置里改了「每周起始日」或语言，TinyCal 会生成新的缓存文件
- TinyCal 更新后缓存被清空了

### 自动运行（可选）

用 launchd 每周一上午 10 点自动运行一次：

```bash
cat > ~/Library/LaunchAgents/local.tinycal-holidays.plist <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>local.tinycal-holidays</string>
  <key>ProgramArguments</key>
  <array><string>$HOME/.local/bin/tinycal-holidays</string></array>
  <key>StartCalendarInterval</key>
  <dict><key>Weekday</key><integer>1</integer><key>Hour</key><integer>10</integer></dict>
</dict>
</plist>
EOF
launchctl load ~/Library/LaunchAgents/local.tinycal-holidays.plist
```

## 原理

TinyCal 的节假日数据是在 `TinyKit.framework` 里写死的 `NSDictionary`，键是 `y2013` 到 `y2022`，从 2023 年起就没有数据了。

不过 TinyCal 计算完每个月的日历后，会把结果存成 plist 缓存：

```
~/Library/Containers/app.cyan.tinycalx/Data/Documents/calendars/
├── 2026.9.0 (zh_CN)      # <年>.<月>.<每周起始日> (<语言>)
├── 2026.10.0 (zh_CN)
└── ...
```

读取顺序是：内存缓存，然后磁盘缓存，最后才现场计算。**只要磁盘缓存存在，TinyCal 就直接使用。** 每个月的缓存里有 42 天的数据（6 行 × 7 列），每一天有一个 `worktime` 字段：

| `worktime` | 含义 | 显示 |
|:-:|---|:-:|
| `0` | 普通日期 | — |
| `1` | 调休上班 | 班 |
| `2` | 法定放假 | 休 |

这个对应关系是从 TinyKit 的反汇编里确认的：`worktime == 1` 时显示 `workday`，`== 2` 时显示 `holiday`。脚本做的事情就是按官方数据改写这个字段，然后重启 TinyCal，清掉它的内存缓存。

## 注意事项

- 只支持**中国大陆**的法定节假日。
- 只处理已经拿到官方数据的年份，其他年份的缓存保持原样。
- 需要联网访问 jsDelivr 或 GitHub。
- 测试环境：TinyCal 1.17.5（App Store 版，Bundle ID `app.cyan.tinycalx`），macOS 13。其他版本如果缓存路径或格式不同，可能用不了。

## 致谢

- [TinyCal](https://apps.apple.com/cn/app/id1114272557)，作者 [@cyanzhong](https://github.com/cyanzhong)
- [NateScarlet/holiday-cn](https://github.com/NateScarlet/holiday-cn)，提供中国法定节假日数据

## License

[MIT](LICENSE)。本项目是非官方工具，与 TinyCal 作者没有关系，使用风险自负。
