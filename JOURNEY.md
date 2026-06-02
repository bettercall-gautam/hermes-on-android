# The Hermes Journey 🧭

> How I built a 24/7 personal AI agent on an old Android phone, with zero budget, zero prior experience, and a painful number of things going wrong along the way.

---

## Where it started

I am lazy. Genuinely, productively lazy. The kind of lazy that makes you spend 10 hours automating something that would have taken 2 hours to do manually, just so you never have to do it again.

I wanted an AI agent that could handle my emails, manage my GitHub, and eventually take over every repetitive task in my life. Something always running, always available, always ready when I messaged it.

The problem? Every solution I found either cost money, required a powerful server, or needed some cloud subscription I could not afford as a student.

Then I looked at my drawer. Sitting there was a Samsung Galaxy A04e, 3GB RAM, Android 14, doing absolutely nothing. Just collecting dust like every side project I have ever started.

So I thought, what if that phone became my server?

That thought started everything.

---

## Phase 1: Getting Termux to work

The first thing I learned is that you should never install Termux from the Play Store. The Play Store version is abandoned, outdated, and will break your packages silently. Always install from F-Droid.

Once I had Termux running, the first real problem hit immediately.

**pkg update got stuck at 3%.**

Termux had automatically picked a slow mirror. The fix was simple once I knew it, run `termux-change-repo` and switch to an Asia region mirror. But nobody tells you this upfront.

After that, the upgrade process kept stopping to ask about config files. openssl.cnf, sources.list, bash.bashrc, one after another. The answer every single time is N, keep your current version. This is not documented anywhere clearly either.

Small things, but they stop a complete beginner cold.

---

## Phase 2: Installing Hermes and the first provider disaster

Installing Hermes itself was actually smooth. One curl command and it handled everything.

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

The real pain started when I tried to connect an AI provider.

My first attempt was **DeepSeek**. HTTP 402, insufficient balance. The free credits that were supposed to apply to my account simply did not. Dead end.

Then I tried **Groq**. Groq looked perfect on paper, fast models, generous free tier. Except Hermes sends an enormous amount of data with every single request. Your SOUL.md, USER.md, memory database, toolsets, and the entire conversation history go with every message. My combined payload was over 18,000 tokens. Groq's limit was 6,000 TPM. It rejected every message with a wall of errors 30 lines long.

I trimmed every file aggressively. Disabled toolsets. Disabled memory. Still too large.

Groq was out.

Then I went to **Gemini**. It worked. Finally something worked. I celebrated for about 20 minutes before hitting the free tier limit.

20 requests per day. That is it. Every article I found said 1500 RPD. Both accounts I tried showed 20 RPD. Every article was wrong. This is just India's reality on the free tier and no account trick fixes it.

I even created a fresh Google account. Also 20 RPD. Same limit. Different account. Same pain.

---

## Phase 3: The provider that actually worked

After all of that, I found the real solution: **OpenRouter**.

OpenRouter acts as a router across multiple free AI models. Instead of being locked to one provider with one limit, it spreads requests across whichever free model is available. The cost is exactly $0.00.

One important warning I learned the hard way: do not use `openrouter/auto` as your model. That setting silently picks paid models without telling you. Use `openrouter/free` which forces it to only route to free models.

I then built a proper fallback chain so Hermes never fully dies even when one provider hits its limit:

1. **OpenRouter free** as primary, handles most requests
2. **Gemini 2.5 Flash** as first fallback, 20 RPD but stable
3. **Cerebras llama-3.3-70b** as last resort

Combined this gives around 220 plus free requests per day. Enough for a real daily driver.

One more thing: Hermes will confidently tell you how many requests you have used. Do not trust these numbers. It hallucinates usage stats with complete confidence. Always check your real usage at openrouter.ai/activity directly.

---

## Phase 4: SOUL.md and the great wipeouts

SOUL.md is the personality file. It tells Hermes how to behave, how to talk, what to do and what never to do. Getting this right took longer than anything else.

The first problem was that the default config had `personality: kawaii` set, which was silently overriding everything I put in SOUL.md. Hermes was ignoring the entire file and acting kawaii instead. Fixed with one sed command but took a while to diagnose.

The bigger problem was that SOUL.md kept getting wiped.

Not edited. Not corrupted. Wiped. Completely replaced with 3 lines. Then completely replaced with just formatting rules. Two different AI providers did this in the same session, on separate occasions, with complete confidence, and zero remorse.

The lesson here is critical and I will say it clearly: **never tell your AI agent to update or rewrite SOUL.md**. Always say append only. Always verify the full content after any change by asking Hermes to show you the complete file. If you do not verify, you will not know it was wiped until Hermes starts behaving completely wrong.

We rebuilt SOUL.md three times in total before adding explicit SOUL.md protection rules inside the file itself.

---

## Phase 5: Connecting Gmail

Gmail was the integration I wanted most. Being able to tell Hermes to check my inbox or draft a reply over Telegram felt like actual magic.

The setup uses Google OAuth2 which sounds scary but is actually straightforward once you know the steps.

You create a Google Cloud project, enable the Gmail API, create OAuth2 credentials, download the client secret JSON, run the auth script, authorize in your browser, and you are done. The token saves automatically.

The working test command:

```bash
python3 ~/.hermes/skills/productivity/google-workspace/scripts/google_api.py gmail search "is:unread" --max 5
```

When this shows your actual unread emails in the terminal, Gmail is connected. It is a genuinely satisfying moment after everything that came before it.

---

## Phase 6: GitHub and the hidden skills folder

Connecting GitHub was the easiest integration of everything. Generate a Personal Access Token from GitHub settings with repo permissions, add it to Hermes, done.

What I did not know until mid-session was that Hermes already had a full GitHub skills folder built in from the start. We almost went down the path of installing external libraries for nothing. Always check what Hermes already has before adding anything new.

---

## Phase 7: Auto start on reboot

The whole point of running this on a dedicated phone is that it should just work, always, without me having to open Termux and manually start anything.

Termux Boot handles this. Install it from F-Droid, create a script at `~/.termux/boot/start-hermes.sh`, make it executable, and Hermes starts automatically every time the phone reboots.

The script also sends a Telegram notification when Hermes comes online and another one when it shuts down. So I always know the status of my agent from my main phone without touching anything.

One thing Samsung One UI does aggressively is kill background processes to save battery. You must set battery optimization to Unrestricted for both Termux and Termux Boot in phone settings. If you skip this, Android will quietly murder your agent in the background and you will wonder why it stopped responding.

---

## Where things stand now

Hermes is running 24/7 on the Samsung A04e. It starts automatically on reboot, sends me a Telegram notification when it comes online, responds to natural language commands, reads and sends emails via Gmail, manages my GitHub, and costs exactly $0.00 per month to operate.

**What is connected and working:**
- Telegram interface
- OpenRouter with Gemini and Cerebras fallback chain
- Gmail via OAuth2
- GitHub via PAT
- Auto start on reboot
- Custom personality via SOUL.md
- Personal context via USER.md

**What is coming next:**
- Google Calendar integration
- Reddit integration
- More automation tools as I find things I want to stop doing manually

---

## Lessons learned

After all of this, here is what I would tell someone starting from zero:

**Never use the Play Store version of Termux.** F-Droid only, always.

**Build a fallback chain from day one.** Single provider setups will fail. A chain of three free providers gives you enough daily requests to actually use the agent.

**Never trust what Hermes says about its own usage numbers.** Always verify at the source.

**SOUL.md is sacred. Protect it.** Append only, verify after every change, never replace.

**Check what skills Hermes already has** before installing anything extra. It ships with more than you think.

**Set battery optimization to Unrestricted** for Termux and Termux Boot or Android will kill your agent silently.

**Start simple.** Get it running first. Connect tools one at a time. Debug one thing at a time. The biggest mistakes happened when I tried to configure too many things at once.

---

## Why I am sharing this

Because when I started, I could not find a single guide that covered running a personal AI agent on Android with zero budget. Every tutorial assumed you had a VPS, a paid API subscription, or at minimum a decent PC.

This entire setup costs nothing. It runs on hardware most people have lying around. And it actually works.

If this helped you build your own Hermes, drop a star on the repo. And if something in this guide is wrong or outdated, open an issue. This is a living project and it gets updated as the agent gets more capable.

The full setup guide is in [README.md](./README.md).

---

*Built with zero budget. Running 24/7 on a phone that was about to be sold for ₹3000.*
