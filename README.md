import discord
from discord.ext import commands, tasks
import asyncio
import json
import os
import openai
from datetime import datetime, timedelta

# ------------- CONFIG -------------
BOT_PREFIX = "!"
intents = discord.Intents.default()
intents.message_content = True
bot = commands.Bot(command_prefix=BOT_PREFIX, intents=intents)

DISCORD_TOKEN = os.getenv("DISCORD_BOT_TOKEN")
openai.api_key = os.getenv("OPENAI_API_KEY")

SCORES_FILE = "scores.json"
active_quizzes = {}
DAILY_CHANNEL_ID = int(os.getenv("DAILY_CHANNEL_ID", "0"))  # set this in .env
DAILY_TIME_UTC = os.getenv("DAILY_TIME_UTC", "14:00")  # e.g., 14:00

LOGO_URL = "https://raw.githubusercontent.com/<your-username>/ritual-quiz-bot/main/assets/ritual_logo.png"

# ------------- SCORE UTILS -------------
def load_scores():
    if not os.path.exists(SCORES_FILE):
        return {}
    with open(SCORES_FILE, "r") as f:
        return json.load(f)

def save_scores(data):
    with open(SCORES_FILE, "w") as f:
        json.dump(data, f, indent=2)

# ------------- AI QUESTION GENERATOR -------------
async def generate_ritual_questions(n=5):
    prompt = f"""
    Generate {n} advanced multiple-choice quiz questions about:
    - Ritual network
    - AI agents on-chain
    - Decentralized compute
    - Web3 infrastructure
    - AI ethics and coordination
    Format output as JSON array, each like:
    {{
      "question": "...",
      "choices": ["...", "...", "...", "..."],
      "answer_index": 0
    }}
    """

    try:
        response = await openai.ChatCompletion.acreate(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )
        content = response.choices[0].message.content
        return json.loads(content)
    except Exception as e:
        print("AI fetch error:", e)
        return []

# ------------- QUIZ UI (BUTTONS) -------------
class QuizView(discord.ui.View):
    def __init__(self, ctx, question_data, correct_label):
        super().__init__(timeout=15)
        self.ctx = ctx
        self.question_data = question_data
        self.correct_label = correct_label
        self.answers = {}

    @discord.ui.button(label="A", style=discord.ButtonStyle.primary)
    async def btn_a(self, i: discord.Interaction, b: discord.ui.Button):
        await self.process_answer(i, "A")

    @discord.ui.button(label="B", style=discord.ButtonStyle.primary)
    async def btn_b(self, i: discord.Interaction, b: discord.ui.Button):
        await self.process_answer(i, "B")

    @discord.ui.button(label="C", style=discord.ButtonStyle.primary)
    async def btn_c(self, i: discord.Interaction, b: discord.ui.Button):
        await self.process_answer(i, "C")

    @discord.ui.button(label="D", style=discord.ButtonStyle.primary)
    async def btn_d(self, i: discord.Interaction, b: discord.ui.Button):
        await self.process_answer(i, "D")

    async def process_answer(self, i, choice):
        uid = str(i.user.id)
        if uid in self.answers:
            await i.response.send_message("❗ You already answered.", ephemeral=True)
            return
        self.answers[uid] = choice
        await i.response.send_message(f"✅ Locked answer: `{choice}`", ephemeral=True)

    async def on_timeout(self):
        await reveal_answer(self.ctx, self.question_data, self.correct_label, self.answers)

# ------------- COMMANDS -------------
@bot.event
async def on_ready():
    print(f"✅ Ritual Quiz Bot active as {bot.user}")
    schedule_daily_quiz.start()

@bot.command(name="startquiz")
async def start_quiz(ctx, n: int = 5):
    await run_ritual_quiz(ctx, n)

async def run_ritual_quiz(ctx, n):
    await ctx.send("🔮 Generating Ritual questions...")
    questions = await generate_ritual_questions(n)
    if not questions:
        await ctx.send("⚠️ AI failed to fetch questions.")
        return

    active_quizzes[ctx.channel.id] = {"questions": questions, "index": 0}
    await ask_next_question(ctx)

async def ask_next_question(ctx):
    state = active_quizzes[ctx.channel.id]
    idx = state["index"]
    questions = state["questions"]

    if idx >= len(questions):
        await end_quiz(ctx)
        return

    q = questions[idx]
    correct_label = ["A", "B", "C", "D"][q["answer_index"]]

    embed = discord.Embed(
        title=f"Ritual Quiz • Question {idx+1}/{len(questions)}",
        description=q["question"],
        color=0x00C3FF
    )
    embed.set_thumbnail(url=LOGO_URL)
    for i, opt in enumerate(q["choices"]):
        embed.add_field(name=["A","B","C","D"][i], value=opt, inline=False)
    embed.set_footer(text="Click the correct answer below. ⏱ 15s")

    view = QuizView(ctx, q, correct_label)
    await ctx.send(embed=embed, view=view)

async def reveal_answer(ctx, q, correct_label, answers):
    state = active_quizzes[ctx.channel.id]
    scores = load_scores()
    correct_idx = q["answer_index"]
    correct_text = q["choices"][correct_idx]
    correct_users = []

    for uid, ans in answers.items():
        if ans == correct_label:
            scores[uid] = scores.get(uid, 0) + 1
            correct_users.append(int(uid))

    save_scores(scores)
    msg = f"✅ **Correct:** {correct_label} - {correct_text}\n"
    msg += "🎉 " + " ".join(f"<@{u}>" for u in correct_users) if correct_users else "😅 No one got it right."
    await ctx.send(msg)
    state["index"] += 1
    await asyncio.sleep(3)
    await ask_next_question(ctx)

@bot.command(name="leaderboard")
async def leaderboard(ctx):
    scores = load_scores()
    if not scores:
        await ctx.send("No scores yet.")
        return
    top = sorted(scores.items(), key=lambda kv: kv[1], reverse=True)[:10]
    embed = discord.Embed(title="🏆 Ritual Leaderboard", color=0xFFD700)
    for i, (uid, pts) in enumerate(top, 1):
        embed.add_field(name=f"{i}.", value=f"<@{uid}> — {pts} pts", inline=False)
    await ctx.send(embed=embed)

# ------------- DAILY QUIZ SCHEDULER -------------
@tasks.loop(minutes=1)
async def schedule_daily_quiz():
    if not DAILY_CHANNEL_ID:
        return
    now = datetime.utcnow().strftime("%H:%M")
    if now == DAILY_TIME_UTC:
        channel = bot.get_channel(DAILY_CHANNEL_ID)
        if channel:
            await channel.send("🕊 Starting Daily Ritual Quiz!")
            await run_ritual_quiz(channel, 5)
            await asyncio.sleep(60)  # avoid multiple triggers same minute

async def end_quiz(ctx):
    del active_quizzes[ctx.channel.id]
    await ctx.send("✨ Ritual Quiz complete! Use `!leaderboard` to view scores.")

# ------------- RUN -------------
if __name__ == "__main__":
    bot.run(DISCORD_TOKEN)
