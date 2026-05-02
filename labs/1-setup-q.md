# Setting Up Amazon Q Developer

**Estimated time:** 15 minutes  
**Required:** A computer with VS Code or IntelliJ installed

---

## What You Need Before Starting

- [ ] VS Code (version 1.85.0 or later) **or** a JetBrains IDE (version 2024.3 or later)
- [ ] An internet connection
- [ ] An email address to create an AWS Builder ID (free — no credit card required)

---

## Step 1 — Create an AWS Builder ID

Amazon Q Developer is free to use with an AWS Builder ID. You do not need an AWS account or a credit card.

1. Go to [https://profile.aws.amazon.com](https://profile.aws.amazon.com)
2. Click **Create an AWS Builder ID**
3. Enter your email address and click **Next**
4. Enter a display name (your name or username)
5. Click **Next** — AWS will send a verification code to your email
6. Enter the code and click **Verify**
7. Set a password and click **Create AWS Builder ID**

Keep this browser tab open — you will need it during sign-in in Step 3.

---

## Step 2 — Install the Amazon Q Extension

### Option A — VS Code

1. Open VS Code
2. Click the **Extensions** icon in the left sidebar (or press `Ctrl+Shift+X` / `Cmd+Shift+X`)
3. Search for **Amazon Q**
4. Click **Install** on the extension published by **Amazon Web Services**
5. Wait for the installation to complete — the Amazon Q icon (a chat bubble with a Q) will appear in your left sidebar

### Option B — IntelliJ (or any JetBrains IDE)

1. Open IntelliJ IDEA
2. Go to **File → Settings → Plugins** (Windows/Linux) or **IntelliJ IDEA → Settings → Plugins** (macOS)
3. Click the **Marketplace** tab
4. Search for **Amazon Q**
5. Click **Install** next to the Amazon Q plugin published by Amazon Web Services
6. Click **Restart IDE** when prompted

---

## Step 3 — Sign In with Your Builder ID

### In VS Code

1. Click the **Amazon Q icon** in the left sidebar — a panel opens at the bottom
2. Click **Sign in to get started**
3. Select **Use for free with Builder ID**
4. Your browser will open — log in with the Builder ID you created in Step 1
5. Click **Allow** to grant the IDE access
6. Return to VS Code — you should see the Amazon Q chat panel is now active

### In IntelliJ

1. Click the **Amazon Q icon** in the right-side toolbar (or go to **View → Tool Windows → Amazon Q**)
2. Click **Sign In**
3. Select **AWS Builder ID**
4. Your browser opens — log in and click **Allow**
5. Return to IntelliJ — the Q panel is now active

> **Sign-in session duration:** Sessions authenticated with Builder ID last 90 days before re-authentication is needed.

---

## Step 4 — Verify the Installation

Test that Amazon Q is working by opening the chat panel and asking a question.

**In VS Code:** Click the Amazon Q icon in the sidebar to open the chat panel.  
**In IntelliJ:** Click the Amazon Q icon or press the Q button in the bottom toolbar.

Type the following and press Enter:
```
What Java version introduced records?
```

Amazon Q should respond with an explanation of Java 16/17 records. If you get a response, the installation is working correctly.

---

## Step 5 — Try Inline Code Suggestions

Amazon Q provides real-time code completions as you type.

1. Open any existing Java (or Python, TypeScript, etc.) file in your project
2. Start typing a method — for example:

```java
public List<User> findActiveUsers
```

3. Pause for 1–2 seconds — Amazon Q will suggest a completion in grey text
4. Press **Tab** to accept the suggestion, or keep typing to dismiss it

If suggestions do not appear, check that the extension is enabled:
- **VS Code:** Check the status bar at the bottom — it should show **Amazon Q** with no error icon
- **IntelliJ:** Go to **Settings → Tools → Amazon Q** and confirm **Enable Amazon Q** is checked

---

## Step 6 — Use the /dev Agentic Mode

The `/dev` command activates agentic mode — the agent reads your project, writes files, and runs builds on your behalf.

1. Open the Amazon Q chat panel
2. Type `/dev` followed by a space and your instruction:

```
/dev Add a health check endpoint at GET /health that returns {"status": "ok"}
```

3. Press Enter and watch the agent:
   - Read your project structure
   - Plan what files to create or modify
   - Write the code
   - Run the build to verify it compiles

> **Note:** `/dev` is available on the Free Tier with a monthly limit of 10 invocations. It is unlimited on the Pro Tier ($19/month per user).

---

## Tier Comparison

| Feature | Free Tier | Pro Tier |
|---|---|---|
| Inline code suggestions | ✅ Unlimited | ✅ Unlimited |
| Chat (Q&A about code) | ✅ Unlimited | ✅ Unlimited |
| `/dev` agentic mode | ✅ 10/month | ✅ Unlimited |
| Security scanning | ✅ Limited | ✅ Unlimited |
| Authentication | AWS Builder ID | IAM Identity Center |

**For this training program:** The Free Tier is sufficient for all Day 1 labs (10 `/dev` invocations covers all lab exercises).

---

## Troubleshooting

**The Amazon Q icon does not appear after installation**  
Restart your IDE fully (not just reload window). In VS Code, close all windows and reopen from the terminal with `code .`

**Sign-in browser tab opens but returns an error**  
Make sure you are logged into your Builder ID in the browser before starting the sign-in flow in the IDE. Try signing in at [profile.aws.amazon.com](https://profile.aws.amazon.com) first, then retry the IDE sign-in.

**Inline suggestions are not appearing**  
Check that the file type is supported. Amazon Q supports Java, Python, JavaScript, TypeScript, C#, Go, Rust, PHP, Ruby, Kotlin, C, C++, SQL, Scala, and others. Plain text files do not trigger suggestions.

**`/dev` command is not recognised**  
Type `/dev` with a space after it before your instruction. Also confirm you are signed in — unauthenticated users cannot access agentic mode.

**IAM credentials do not work for sign-in**  
Amazon Q in IDEs does not support IAM-based sign-in. Use AWS Builder ID (free) or IAM Identity Center (Pro tier) only.

---

## What's Next

Once Amazon Q is installed and signed in, you are ready for:

- **Lab 1** — Use `/dev` to implement your first feature from a single prompt
- **Lab 2** — Author steering files that give the agent persistent knowledge of your project
- **Lab 3** — Run the spec workflow to generate requirements, architecture, and tasks before writing any code