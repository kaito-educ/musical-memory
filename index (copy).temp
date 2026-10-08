const { Telegraf, Markup } = require("telegraf");
const axios = require("axios");
const fs = require("fs");
const path = require("path");
const os = require("os");
const ffmpeg = require("fluent-ffmpeg");
const ffmpegPath = require("ffmpeg-static");

// ─────────────────────────────────────────────
// CONFIG
// ─────────────────────────────────────────────

const config = require("./config.json");

if (!config.botToken) {
    console.error("❌ botToken is missing from config.json");
    process.exit(1);
}

if (!config.tikwmApi) {
    console.error("❌ tikwmApi is missing from config.json");
    process.exit(1);
}

const bot = new Telegraf(config.botToken);

ffmpeg.setFfmpegPath(ffmpegPath);

// ─────────────────────────────────────────────
// SETTINGS
// ─────────────────────────────────────────────

const MAX_FILE_SIZE = 49 * 1024 * 1024;

const TEMP_DIR = path.join(
    os.tmpdir(),
    "tiktok-telegram-bot"
);

if (!fs.existsSync(TEMP_DIR)) {
    fs.mkdirSync(TEMP_DIR, {
        recursive: true
    });
}

// ─────────────────────────────────────────────
// TIKTOK URL CHECK
// ─────────────────────────────────────────────

function isTikTokUrl(text) {
    return /^https?:\/\/(?:www\.|vm\.|vt\.|m\.)?tiktok\.com\/\S+/i.test(
        text.trim()
    );
}

// ─────────────────────────────────────────────
// GET TIKTOK DATA FROM TIKWM
// ─────────────────────────────────────────────

async function getTikTokInfo(url) {
    const body = new URLSearchParams();

    body.append("url", url);
    body.append("hd", "1");

    const response = await axios.post(
        config.tikwmApi,
        body.toString(),
        {
            headers: {
                "Content-Type":
                    "application/x-www-form-urlencoded"
            },

            timeout: 30000
        }
    );

    if (
        !response.data ||
        response.data.code !== 0
    ) {
        throw new Error(
            response.data?.msg ||
            "TikWM could not process this video."
        );
    }

    return response.data.data;
}

// ─────────────────────────────────────────────
// INFORMATION HELPERS
// ─────────────────────────────────────────────

function getTitle(data) {
    return data.title || "TikTok Video";
}

function getAuthor(data) {
    return (
        data.author?.nickname ||
        data.author?.unique_id ||
        data.author?.uniqueId ||
        "Unknown"
    );
}

function getUsername(data) {
    return (
        data.author?.unique_id ||
        data.author?.uniqueId ||
        ""
    );
}

function getThumbnail(data) {
    return (
        data.cover ||
        data.origin_cover ||
        null
    );
}

// ─────────────────────────────────────────────
// GET VIDEO URL
// ─────────────────────────────────────────────

function getVideoUrl(data, hd = false) {

    // HD from TikWM
    if (hd) {
        return data.hdplay || null;
    }

    // Normal no-watermark video
    return data.play || null;
}

// ─────────────────────────────────────────────
// SAFE FILE NAME
// ─────────────────────────────────────────────

function cleanFilename(name) {

    return String(name || "tiktok")
        .replace(
            /[<>:"/\\|?*\x00-\x1F]/g,
            ""
        )
        .replace(/\s+/g, " ")
        .trim()
        .slice(0, 80);
}

// ─────────────────────────────────────────────
// DOWNLOAD FILE
// ─────────────────────────────────────────────

async function downloadFile(
    url,
    output,
    onProgress
) {

    const response = await axios.get(
        url,
        {
            responseType: "stream",

            timeout: 120000,

            maxRedirects: 5
        }
    );

    const total = Number(
        response.headers["content-length"] || 0
    );

    if (
        total &&
        total > MAX_FILE_SIZE
    ) {
        throw new Error(
            "The video is too large for this bot."
        );
    }

    return new Promise(
        (resolve, reject) => {

            let downloaded = 0;

            const writer =
                fs.createWriteStream(output);

            response.data.on(
                "data",
                chunk => {

                    downloaded +=
                        chunk.length;

                    if (total > 0) {

                        const percent =
                            Math.floor(
                                (downloaded /
                                    total) *
                                    100
                            );

                        onProgress(percent);
                    }

                    // Safety check while downloading
                    if (
                        downloaded >
                        MAX_FILE_SIZE
                    ) {
                        response.data.destroy();

                        writer.destroy();

                        reject(
                            new Error(
                                "File exceeded the maximum size."
                            )
                        );
                    }
                }
            );

            response.data.on(
                "error",
                reject
            );

            writer.on(
                "error",
                reject
            );

            writer.on(
                "finish",
                () => resolve(output)
            );

            response.data.pipe(writer);
        }
    );
}

// ─────────────────────────────────────────────
// CONVERT VIDEO TO MP3
// ─────────────────────────────────────────────

function convertToMp3(
    input,
    output
) {

    return new Promise(
        (resolve, reject) => {

            ffmpeg(input)
                .audioCodec(
                    "libmp3lame"
                )

                .audioBitrate("128k")

                .format("mp3")

                .on(
                    "error",
                    reject
                )

                .on(
                    "end",
                    resolve
                )

                .save(output);
        }
    );
}

// ─────────────────────────────────────────────
// CLEANUP
// ─────────────────────────────────────────────

function cleanup(files = []) {

    for (const file of files) {

        try {

            if (
                file &&
                fs.existsSync(file)
            ) {
                fs.unlinkSync(file);
            }

        } catch (error) {

            console.error(
                "Cleanup error:",
                error.message
            );
        }
    }
}

// ─────────────────────────────────────────────
// AUTOMATIC CLEANUP
// ─────────────────────────────────────────────

setInterval(
    () => {

        try {

            const files =
                fs.readdirSync(
                    TEMP_DIR
                );

            for (
                const file of files
            ) {

                const fullPath =
                    path.join(
                        TEMP_DIR,
                        file
                    );

                const stat =
                    fs.statSync(
                        fullPath
                    );

                // Delete files older than 30 minutes
                if (
                    Date.now() -
                        stat.mtimeMs >
                    30 * 60 * 1000
                ) {

                    fs.unlinkSync(
                        fullPath
                    );
                }
            }

        } catch (error) {

            console.error(
                "Automatic cleanup:",
                error.message
            );
        }

    },
    10 * 60 * 1000
);

// ─────────────────────────────────────────────
// PROGRESS BAR
// ─────────────────────────────────────────────

function progressBar(
    percent
) {

    const total = 10;

    const filled =
        Math.round(
            (percent / 100) *
                total
        );

    return (
        "█".repeat(filled) +
        "░".repeat(
            total - filled
        )
    );
}

// ─────────────────────────────────────────────
// SAFE TELEGRAM EDIT
// ─────────────────────────────────────────────

async function updateProgress(
    ctx,
    messageId,
    text
) {

    try {

        await ctx.telegram.editMessageText(
            ctx.chat.id,
            messageId,
            undefined,
            text
        );

    } catch {
        // Ignore "message not modified"
    }
}

// ─────────────────────────────────────────────
// /START
// ─────────────────────────────────────────────

bot.start(
    async ctx => {

        await ctx.reply(
            `👋 Welcome to TikTok Downloader!

🎬 Send me a TikTok link.

I'll show you:

🖼️ Thumbnail
📝 Video title
👤 Creator
🎥 Normal Video
✨ HD Video
🎵 MP3 Audio

Paste your TikTok link below.

⚠️ Only download content you own or have permission to use.`,

            Markup.inlineKeyboard([
                [
                    Markup.button.callback(
                        "📖 Help",
                        "HELP"
                    )
                ]
            ])
        );
    }
);

// ─────────────────────────────────────────────
// /HELP
// ─────────────────────────────────────────────

bot.help(
    async ctx => {

        await ctx.reply(
            `📖 HOW TO USE

1️⃣ Copy a TikTok link
2️⃣ Send it to the bot
3️⃣ Wait for the preview
4️⃣ Choose:

🎥 Video
✨ HD Video
🎵 MP3

Commands:

/start
/help
/video <TikTok URL>
/mp3 <TikTok URL>
/hd <TikTok URL>

⚠️ Only download content you have permission to use.`
        );
    }
);

// ─────────────────────────────────────────────
// HELP BUTTON
// ─────────────────────────────────────────────

bot.action(
    "HELP",
    async ctx => {

        await ctx.answerCbQuery();

        await ctx.reply(
            `📖 HOW TO USE

Send a TikTok link to this bot.

Then select:

🎥 Video
Normal video quality

✨ HD Video
Uses TikWM's HD URL

🎵 MP3
Extracts the audio

Commands:

/video <link>
/hd <link>
/mp3 <link>`
        );
    }
);

// ─────────────────────────────────────────────
// /VIDEO
// ─────────────────────────────────────────────

bot.command(
    "video",
    async ctx => {

        const url =
            ctx.message.text
                .replace(
                    /^\/video\s*/i,
                    ""
                )
                .trim();

        if (
            !isTikTokUrl(url)
        ) {

            return ctx.reply(
                "❌ Usage:\n\n/video <TikTok URL>"
            );
        }

        await downloadTikTok(
            ctx,
            url,
            "video"
        );
    }
);

// ─────────────────────────────────────────────
// /HD
// ─────────────────────────────────────────────

bot.command(
    "hd",
    async ctx => {

        const url =
            ctx.message.text
                .replace(
                    /^\/hd\s*/i,
                    ""
                )
                .trim();

        if (
            !isTikTokUrl(url)
        ) {

            return ctx.reply(
                "❌ Usage:\n\n/hd <TikTok URL>"
            );
        }

        await downloadTikTok(
            ctx,
            url,
            "hd"
        );
    }
);

// ─────────────────────────────────────────────
// /MP3
// ─────────────────────────────────────────────

bot.command(
    "mp3",
    async ctx => {

        const url =
            ctx.message.text
                .replace(
                    /^\/mp3\s*/i,
                    ""
                )
                .trim();

        if (
            !isTikTokUrl(url)
        ) {

            return ctx.reply(
                "❌ Usage:\n\n/mp3 <TikTok URL>"
            );
        }

        await downloadTikTok(
            ctx,
            url,
            "mp3"
        );
    }
);

// ─────────────────────────────────────────────
// PROCESS TIKTOK LINK
// ─────────────────────────────────────────────

async function processTikTok(
    ctx,
    url
) {

    const loading =
        await ctx.reply(
            "🔎 Fetching TikTok information..."
        );

    try {

        const data =
            await getTikTokInfo(
                url
            );

        const title =
            getTitle(data);

        const author =
            getAuthor(data);

        const username =
            getUsername(data);

        const thumbnail =
            getThumbnail(data);

        const caption =
            `🎬 ${title}

👤 ${author}` +
            (
                username
                    ? ` (@${username})`
                    : ""
            ) +
            `

👇 Choose what you want to download:`;

        const keyboard =
            Markup.inlineKeyboard([
                [
                    Markup.button.callback(
                        "🎥 Video",
                        `VIDEO:${encodeURIComponent(url)}`
                    ),

                    Markup.button.callback(
                        "✨ HD Video",
                        `HD:${encodeURIComponent(url)}`
                    )
                ],

                [
                    Markup.button.callback(
                        "🎵 MP3",
                        `MP3:${encodeURIComponent(url)}`
                    )
                ]
            ]);

        // Send thumbnail
        if (thumbnail) {

            try {

                await ctx.replyWithPhoto(
                    {
                        url: thumbnail
                    },
                    {
                        caption,
                        ...keyboard
                    }
                );

            } catch {

                await ctx.reply(
                    caption,
                    keyboard
                );
            }

        } else {

            await ctx.reply(
                caption,
                keyboard
            );
        }

        await ctx.telegram.deleteMessage(
            ctx.chat.id,
            loading.message_id
        );

    } catch (error) {

        console.error(
            "TikTok API error:",
            error
        );

        await updateProgress(
            ctx,
            loading.message_id,
            `❌ Could not process this TikTok link.

${error.message}`
        );
    }
}

// ─────────────────────────────────────────────
// VIDEO BUTTON
// ─────────────────────────────────────────────

bot.action(
    /^VIDEO:(.+)$/,
    async ctx => {

        await ctx.answerCbQuery(
            "Preparing video..."
        );

        const url =
            decodeURIComponent(
                ctx.match[1]
            );

        await downloadTikTok(
            ctx,
            url,
            "video"
        );
    }
);

// ─────────────────────────────────────────────
// HD BUTTON
// ─────────────────────────────────────────────

bot.action(
    /^HD:(.+)$/,
    async ctx => {

        await ctx.answerCbQuery(
            "Preparing HD video..."
        );

        const url =
            decodeURIComponent(
                ctx.match[1]
            );

        await downloadTikTok(
            ctx,
            url,
            "hd"
        );
    }
);

// ─────────────────────────────────────────────
// MP3 BUTTON
// ─────────────────────────────────────────────

bot.action(
    /^MP3:(.+)$/,
    async ctx => {

        await ctx.answerCbQuery(
            "Preparing MP3..."
        );

        const url =
            decodeURIComponent(
                ctx.match[1]
            );

        await downloadTikTok(
            ctx,
            url,
            "mp3"
        );
    }
);

// ─────────────────────────────────────────────
// DOWNLOAD
// ─────────────────────────────────────────────

async function downloadTikTok(
    ctx,
    url,
    type
) {

    let inputFile = null;
    let outputFile = null;

    const status =
        await ctx.reply(
            type === "hd"
                ? "✨ Preparing HD video..."
                : type === "video"
                    ? "⬇️ Preparing video..."
                    : "🎵 Preparing MP3..."
        );

    try {

        const data =
            await getTikTokInfo(
                url
            );

        const title =
            cleanFilename(
                getTitle(data)
            );

        const author =
            getAuthor(data);

        // ─────────────────────
        // Select URL
        // ─────────────────────

        let videoUrl;

        if (type === "hd") {

            // IMPORTANT:
            // HD comes directly from
            // TikWM's hdplay field.

            videoUrl =
                data.hdplay;

            if (!videoUrl) {

                throw new Error(
                    "TikWM did not provide an HD video URL for this video."
                );
            }

        } else {

            videoUrl =
                data.play;

            if (!videoUrl) {

                throw new Error(
                    "TikWM did not provide a normal video URL."
                );
            }
        }

        // ─────────────────────
        // Temporary file names
        // ─────────────────────

        const id =
            Date.now() +
            "-" +
            Math.random()
                .toString(36)
                .slice(2);

        inputFile =
            path.join(
                TEMP_DIR,
                `${id}.mp4`
            );

        outputFile =
            path.join(
                TEMP_DIR,
                `${id}.mp3`
            );

        // ─────────────────────
        // DOWNLOAD
        // ─────────────────────

        let lastProgress = -1;

        await downloadFile(
            videoUrl,
            inputFile,
            async percent => {

                if (
                    percent !==
                        lastProgress &&
                    (
                        percent % 10 === 0 ||
                        percent === 100
                    )
                ) {

                    lastProgress =
                        percent;

                    await updateProgress(
                        ctx,
                        status.message_id,

                        `⬇️ Downloading ${
                            type === "hd"
                                ? "HD "
                                : ""
                        }video...

${progressBar(
    percent
)} ${percent}%`
                    );
                }
            }
        );

        // ─────────────────────
        // MP3
        // ─────────────────────

        if (type === "mp3") {

            await updateProgress(
                ctx,
                status.message_id,
                "🎵 Converting video to MP3..."
            );

            await convertToMp3(
                inputFile,
                outputFile
            );

            await updateProgress(
                ctx,
                status.message_id,
                "📤 Uploading MP3..."
            );

            await ctx.replyWithAudio(
                {
                    source: outputFile
                },
                {
                    title: title,
                    performer: author,

                    caption:
                        `🎵 ${title}\n` +
                        `👤 ${author}`
                }
            );

        } else {

            // ─────────────────
            // VIDEO / HD
            // ─────────────────

            await updateProgress(
                ctx,
                status.message_id,
                type === "hd"
                    ? "✨ HD video downloaded!\n\n📤 Uploading..."
                    : "📤 Uploading video..."
            );

            await ctx.replyWithVideo(
                {
                    source: inputFile
                },
                {
                    caption:
                        (
                            type === "hd"
                                ? "✨ HD Video"
                                : "🎥 Video"
                        ) +
                        `\n\n🎬 ${title}` +
                        `\n👤 ${author}`
                }
            );
        }

        // Delete status message
        try {

            await ctx.telegram.deleteMessage(
                ctx.chat.id,
                status.message_id
            );

        } catch {}
    }

    catch (error) {

        console.error(
            "Download error:",
            error
        );

        await updateProgress(
            ctx,
            status.message_id,

            `❌ Download failed.

${error.message}`
        );

    }

    finally {

        // Always delete files
        cleanup([
            inputFile,
            outputFile
        ]);
    }
}

// ─────────────────────────────────────────────
// NORMAL TIKTOK LINKS
// ─────────────────────────────────────────────

bot.on(
    "text",
    async ctx => {

        const text =
            ctx.message.text.trim();

        // Ignore commands
        if (
            text.startsWith("/")
        ) {
            return;
        }

        if (
            !isTikTokUrl(text)
        ) {
            return ctx.reply(
                "❌ Please send a valid TikTok link."
            );
        }

        await processTikTok(
            ctx,
            text
        );
    }
);

// ─────────────────────────────────────────────
// ERROR HANDLER
// ─────────────────────────────────────────────

bot.catch(
    async (error, ctx) => {

        console.error(
            "BOT ERROR:",
            error
        );

        try {

            await ctx.reply(
                "⚠️ Something went wrong. Please try again."
            );

        } catch {}
    }
);

// ─────────────────────────────────────────────
// START BOT
// ─────────────────────────────────────────────

bot.launch();

console.log("");
console.log(
    "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
);
console.log(
    "🤖 TikTok Downloader Bot"
);
console.log(
    "🟢 Bot is running"
);
console.log(
    "✨ Video / HD / MP3 enabled"
);
console.log(
    "🧹 Automatic cleanup enabled"
);
console.log(
    "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
);
console.log("");

// ─────────────────────────────────────────────
// GRACEFUL SHUTDOWN
// ─────────────────────────────────────────────

process.once(
    "SIGINT",
    () => bot.stop("SIGINT")
);

process.once(
    "SIGTERM",
    () => bot.stop("SIGTERM")
);