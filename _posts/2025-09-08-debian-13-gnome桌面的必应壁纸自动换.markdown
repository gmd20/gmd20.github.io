让 AI 写
```bash
#!/bin/bash

# Bing每日图片下载并设置为GNOME桌面背景脚本

# 设置变量
BING_URL="https://www.bing.com"
API_URL="https://www.bing.com/HPImageArchive.aspx?format=js&idx=0&n=1&mkt=en-US"
SAVE_DIR="$HOME/BingWallpapers"
TODAY=$(date +%Y%m%d)
IMAGE_NAME="bing_${TODAY}.jpg"
IMAGE_PATH="$SAVE_DIR/$IMAGE_NAME"

# 创建保存目录
mkdir -p "$SAVE_DIR"

# 获取图片信息
echo "正在获取Bing每日图片信息..."
IMAGE_JSON=$(curl -s "$API_URL")
if [ $? -ne 0 ]; then
    echo "错误: 无法连接到Bing API"
    exit 1
fi

# 解析JSON获取图片URL
IMAGE_URL=$(echo "$IMAGE_JSON" | grep -oP '"url":"[^"]+"' | cut -d'"' -f4)
if [ -z "$IMAGE_URL" ]; then
    echo "错误: 无法解析图片URL"
    exit 1
fi

# 构建完整图片URL
FULL_IMAGE_URL="${BING_URL}${IMAGE_URL}"

# 下载图片
echo "正在下载图片: $FULL_IMAGE_URL"
curl -s -o "$IMAGE_PATH" "$FULL_IMAGE_URL"
if [ $? -ne 0 ]; then
    echo "错误: 图片下载失败"
    exit 1
fi

# 检查图片是否有效
if [ ! -s "$IMAGE_PATH" ]; then
    echo "错误: 下载的图片文件为空"
    rm -f "$IMAGE_PATH"
    exit 1
fi

echo "图片已保存到: $IMAGE_PATH"

# 设置GNOME桌面背景
echo "正在设置GNOME桌面背景..."
gsettings set org.gnome.desktop.background picture-uri "file://$IMAGE_PATH"
gsettings set org.gnome.desktop.background picture-uri-dark "file://$IMAGE_PATH"
gsettings set org.gnome.desktop.background picture-options "zoom"

# 输出成功信息
echo "✅ Bing每日图片已成功设置为桌面背景！"
echo "📁 图片位置: $IMAGE_PATH"

# 可选: 清理 7 天前的旧图片
 find "$SAVE_DIR" -name "bing_*.jpg" -mtime +7 -delete
# echo "已清理 7 天前的旧图片"
```

/etc/systemd/system/bing-wallpaper.service 
```ini
[Unit]
Description=Bing Daily Wallpaper
After=network.target graphical.target

[Service]
Type=oneshot
ExecStart=/home/ming/bing_wallpaper.sh
User=ming
Group=ming
Environment=DISPLAY=:0
Environment=DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus

[Install]
WantedBy=multi-user.target

```

/etc/systemd/system/bing-wallpaper.timer 
```ini
[Unit]
Description=Run Bing Wallpaper daily at 9:00 AM
Requires=bing-wallpaper.service

[Timer]
OnCalendar=*-*-* 09:00:00
Persistent=true

[Install]
WantedBy=timers.target

```

