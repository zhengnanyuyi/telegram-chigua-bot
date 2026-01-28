# -*- coding: utf-8 -*-
import os
import asyncio
import subprocess
import requests
from bs4 import BeautifulSoup
from googletrans import Translator
from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import ApplicationBuilder, CallbackQueryHandler, ContextTypes

# ================== 环境变量 ==================
BOT_TOKEN = os.getenv("BOT_TOKEN")
ADMIN_ID = int(os.getenv("ADMIN_ID"))
CHANNEL_ID = os.getenv("CHANNEL_ID")

# ================== 配置 ==================
NITTER_BASE = "https://nitter.net"
X_USERS = [
    "DramaAlert",
    "PopCrave",
    "Dexerto",
]

FETCH_INTERVAL = 180
sent_cache = set()
translator = Translator()

# ================== 标题生成 ==================
def make_title(text):
    text = text[:80]
    if any(k in text for k in ["exposed", "caught", "leaked"]):
        return "🍉 网红翻车现场"
    if any(k in text for k in ["weird", "wtf", "insane"]):
        return "👀 今日迷惑行为"
    return "😂 吃瓜现场"

# ================== 抓 X ==================
def fetch_x_posts():
    results = []

    for user in X_USERS:
        url = f"{NITTER_BASE}/{user}"
        r = requests.get(url, timeout=15)
        soup = BeautifulSoup(r.text, "html.parser")

        for item in soup.select(".timeline-item"):
            tweet_id = item.get("data-id")
            if not tweet_id or tweet_id in sent_cache:
                continue

            content = item.select_one(".tweet-content")
            if not content:
                continue

            text = content.get_text(strip=True)
            if not text:
                continue

            # 翻译
            zh = translator.translate(text, dest="zh-cn").text
            title = make_title(text)

            # 图片
            images = []
            for img in item.select("a.still-image"):
                img_url = img.get("href")
                if img_url:
                    images.append(NITTER_BASE + img_url)

            # 视频
            video = None
            video_link = item.select_one("a.video-container")
            if video_link:
                video_url = NITTER_BASE + video_link.get("href")
                try:
                    subprocess.run(
                        ["yt-dlp", "-f", "mp4", "-o", "video.mp4", video_url],
                        timeout=40
                    )
                    video = "video.mp4"
                except:
                    pass

            if not images and not video:
                continue

            results.append({
                "id": tweet_id,
                "text": zh,
                "title": title,
                "images": images,
                "video": video
            })
            sent_cache.add(tweet_id)

    return results

# ================== 发送审核 ==================
async def send_for_review(app):
    while True:
        posts = fetch_x_posts()

        for p in posts:
            caption = f"{p['title']}\n\n{p['text']}"

            kb = InlineKeyboardMarkup([
                [
                    InlineKeyboardButton("✅ 发布", callback_data="publish"),
                    InlineKeyboardButton("❌ 丢弃", callback_data="discard"),
                ]
            ])

            if p["video"]:
                await app.bot.send_video(
                    ADMIN_ID,
                    video=open(p["video"], "rb"),
                    caption=caption,
                    reply_markup=kb
                )

            elif p["images"]:
                await app.bot.send_message(ADMIN_ID, caption)
                for img in p["images"]:
                    await app.bot.send_photo(ADMIN_ID, img, reply_markup=kb)

        await asyncio.sleep(FETCH_INTERVAL)

# ================== 审核 ==================
async def review(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q = update.callback_query
    await q.answer()

    if q.data == "publish":
        if q.message.video:
            await context.bot.send_video(
                CHANNEL_ID,
                q.message.video.file_id,
                caption=q.message.caption
            )
        elif q.message.photo:
            await context.bot.send_photo(
                CHANNEL_ID,
                q.message.photo[-1].file_id,
                caption=q.message.caption
            )
        else:
            await context.bot.send_message(
                CHANNEL_ID,
                q.message.text
            )
        await q.edit_message_text("✅ 已发布")
    else:
        await q.edit_message_text("❌ 已丢弃")

# ================== 主入口 ==================
if __name__ == "__main__":
    app = ApplicationBuilder().token(BOT_TOKEN).build()
    app.add_handler(CallbackQueryHandler(review))
    loop = asyncio.get_event_loop()
    loop.create_task(send_for_review(app))
    print("🤖 X 吃瓜 Bot 启动")
    app.run_polling()
