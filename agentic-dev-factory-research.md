# בניית "מפעל" לפיתוח אג'נטי עם Claude Code — מחקר מעמיק

*מחקר שנערך ביוני 2026. מבוסס על fan-out של 5 צירי חיפוש, אימות אדוורסרי של הטענות המרכזיות, וסינתזה ממוקדת-מקורות. כל מספר/טענה נושאת קישור למקור.*

---

## תקציר מנהלים (TL;DR)

1. **העיקרון של IndyDevDan נכון, אבל הניסוח מדויק יותר הוא "build the system that builds the system"** — לבנות תשתית אג'נטית רב-פעמית שעושה עבודה אוטונומית עבורך, במקום להשתמש ב-AI ככלי יד. ("factory not a tool" הוא פרפראזה קהילתית של אותו רעיון.)

2. **"10x" הוא לרוב אשליה ברמת הארגון.** מחקר RCT של METR מצא שמפתחים מנוסים היו **19% איטיים יותר** עם AI — בעוד שהם *הרגישו* 20% מהירים יותר. במקביל סקרים גדולים (DORA 2025) מראים עלייה בתפוקה. שני הדברים נכונים: AI מגדיל תפוקה גולמית אבל **מעביר את צוואר הבקבוק ל-review** (זמן הריוויו עלה ~91%).

3. **הממצא החשוב ביותר: "AI לא מתקן צוות — הוא מגביר את מה שכבר קיים."** צוותים חזקים עם תשתית טובה נעשים יעילים; צוותים חלשים נעשים גרועים יותר. **התשתית היא המכפיל, לא המודל.**

4. **חמשת העמודים שלך נכונים אבל חסרים ~7 קריטיים** — בעיקר: לולאות אימות סגורות (closed-loop), evals פרטיים נגד reward hacking, guardrails ברמת התשתית (לא ברמת ה-prompt), observability+בקרת עלות, תזמור agents מקבילים, ו-CI/CD headless כעמוד השדרה של ה"מפעל".

---

## חלק 1 — מה זה "מפעל ולא כלי" באמת (IndyDevDan)

מי הוא: IndyDevDan (handle ב-GitHub: `disler`), יוצר תוכן וקורסים ("Principled AI Coding", "Tactical Agentic Coding"). הריפוזיטוריז שלו הם המקורות הראשוניים הכי אמינים.

**העקרונות המאומתים שלו:**

- **"build the system that builds the system" / "living software"** — המעבר מ-AI שכותב את שכבת-האפליקציה, לבניית המערכת שבונה אפליקציות. [מאומת verbatim ב-agenticengineer.com]
- **היררכיית עדיפויות: `Agent > Code > Manual Input`** — העדף agent אוטונומי על-פני כתיבת קוד, וקוד על-פני קלט ידני. [github.com/disler/indydevtools]
- **Prompts/agents הם "the new fundamental unit of programming"** — מנוהלים בגרסאות, מובנים (YAML frontmatter + סקשנים קבועים: Purpose, Variables, Codebase Structure, Instructions, Workflow, Report), רב-פעמיים, וניידים בין harnesses. [gist של disler]
- **"big three → core four":** Context, Model, Prompt הם הבסיס; הוספת **Tools** היא מה שהופך את זה ל-agentic coding. [agenticengineer.com snippets]
- **Context engineering ≠ גודל אלא רלוונטיות:** "context is not about size — it's about relevance and focus"; "polluted context... opens yourself up to mistakes."
- **"compute advantage":** "Your value as an engineer scales directly with the amount of compute you can harness" / "the future isn't for those who write code. It's for those who manage computations." [מאומת]
- **אמינות דרך דטרמיניזם, לא רק prompting:** שימוש ב-hooks (PreToolUse/PostToolUse/Stop) ל"deterministic control without relying on LLM decisions". [github.com/disler/claude-code-hooks-mastery]
- **Closed-loop / AFK agents / ZTE:** התקדמות בגרות "in-loop → out-loop → ZTE" (zero touch); agents שמתכננים, מבצעים ומתקנים את עצמם ללא פיקוח.

> **אזהרת ייחוס:** הביטוי המדויק "build a factory, not a tool" לא אומת כציטוט verbatim שלו. הפריימינג "factory" הכי מתועד אצל **Addy Osmani** ("The Factory Model": *"you are no longer just writing code — you are building the factory that builds your software"*). הרעיון הוא של IndyDevDan; הניסוח הספציפי הוא קהילתי.

**המקורות הניתנים להרצה (open source) שלו:** `claude-code-is-programmable`, `single-file-agents`, `the-library`, `claude-code-hooks-mastery`, `indydevtools`.

---

## חלק 2 — המציאות לפי הנתונים: כמה באמת אפשר להאיץ?

### צד ה"hype" (תפוקה עולה)
- **DORA 2025** (~5,000 משיבים): 90% משתמשים ב-AI; אימוץ AI כעת **בקורלציה חיובית** עם throughput של אספקת תוכנה. [cloud.google.com — מקור ראשוני שאומת]
- **טלמטריה (Faros/DORA, 10,000+ מפתחים):** צוותים בשימוש-AI גבוה השלימו ~**21% יותר משימות** ומיזגו ~**98% יותר PRs**. [faros.ai]
- **Anthropic פנימית:** 70–90% מהקוד נכתב בעזרת Claude Code; הקודבייס של Claude Code עצמו ~90% נכתב על-ידו; ~67% יותר PRs/יום. [claude.com/blog]

### צד הראיות הנגדיות (קריטי!)
- **METR RCT (היהלום המתודולוגי):** מפתחים מנוסים על קודבייס מוכר היו **19% איטיים יותר** — למרות שחזו 24% מהירות ו*הרגישו* 20% מהירים. [arXiv 2507.09089, יולי 2025 — **אומת**]
- **GitClear (211M שורות):** קוד מועתק עלה מ-<10% ל-~15%; refactoring ("moved lines") צנח מ-24% (2021) ל-<10%; בלוקים כפולים ×8 ב-2024. [gitclear.com]
- **DORA:** אימוץ AI עדיין ב**קורלציה שלילית עם יציבות** האספקה; 30% לא סומכים על קוד שנוצר ב-AI. [cloud.google.com]
- **צוואר הבקבוק החדש — review:** זמן ריוויו עלה ~**91%**; קוד-AI חשף ×1.7 יותר בעיות ל-PR; מ-10–15 PRs/שבוע ל-50–100. [logrocket, codacy, aviator]
- **ביקורת Amdahl:** אם קוד הוא ~20% ממחזור האספקה, האצה ×10 בקוד נותנת רק ~×1.25 כולל. **10x אמיתי דורש האצה של review, test, deploy, ops.** [augmentcode.com]

### המסקנה המאחדת (הכי חשוב)
> **"AI doesn't fix a team; it amplifies what's already there."** (DORA 2025). 90% מהארגונים בנו פלטפורמה פנימית; **איכות הפלטפורמה הפנימית בקורלציה ישירה עם הצלחת ה-AI.** מי שזורק כלים על תשתית חלשה — מגביר את התפקוד הלקוי.

---

## חלק 3 — ארכיטקטורת ה"מפעל": 5 העמודים שלך + מה שחסר

חמשת העמודים שלך נכונים. הנה הם ממופים לתוך ארכיטקטורה שלמה, עם **7 העמודים החסרים** מסומנים ⭐.

### שכבה 0 — בסיס: הקשר ומפרט
| # | עמוד | סטטוס |
|---|------|--------|
| 1 | **תיעוד/קונטקסט מתעדכן** (שלך) | `CLAUDE.md` (<200 שורות, מנחה לא אוכף), auto-memory (v2.1.59+), `.claude/rules/*.md` עם `paths:` frontmatter לטעינה לפי-קונטקסט. **לא AGENTS.md** — Claude Code קורא CLAUDE.md (גשר: `@AGENTS.md`). |
| 5 | **אפיון עמוק של משימות** (שלך) | זה למעשה **Spec-Driven Development** ⭐. ראה שכבה 4. |

### שכבה 1 — היחידות הרב-פעמיות (לב ה"מפעל")
| # | עמוד | פירוט |
|---|------|--------|
| 2 | **מאגר skills** (שלך) | `SKILL.md` נטען רק בשימוש (progressive disclosure) — "זול עד שצריך". Commands אוחדו לתוך skills. בנו ספרייה משותפת (case study: 22 skills עד יום 90, 9 מהנדסים). |
| ⭐6 | **Subagents + MCP + Hooks** | ארבעת ה-primitives נפרדים: **Subagents** = בידוד-קונטקסט + הגבלת tools/model (חסכון עלות). **MCP** = חיבור למערכות חיצוניות (Jira, Postgres, Sentry). **Hooks** = אוטומציה דטרמיניסטית ב-lifecycle (30+ אירועים). הם מתחברים: plugin יכול לארוז את כולם. |

### שכבה 2 — הלולאה הסגורה ושערי האיכות (⭐ החוסר המרכזי)
| # | עמוד | פירוט |
|---|------|--------|
| 3 | **טסטים איכותיים** (שלך) | נכון — אבל ראה "reward hacking" למטה: "טסטים עוברים" הוא proxy דולף. |
| ⭐7 | **לולאות אימות (closed-loop)** | **"ה-agent הוא הלולאה."** אמינות מגיעה מאיכות ה-ground truth (טסטים + linters + types + compiler), לא מ"מודל חכם יותר". Anthropic: feedback מבוסס-כללים (linting) הוא הסוג הטוב ביותר. |
| ⭐8 | **Evals פרטיים** | בנו eval suite על **הקוד שלכם** — benchmarks ציבוריים מזוהמים (SWE-Bench Verified עם 37 פתרונות דלופים; 70%→23% ב-SWE-Bench Pro). מודדים אם ה-skill/agent שומר על הקונבנציות שלכם. |
| ⭐8b | **הגנה מ-reward hacking** | "טסטים עוברים ≠ הצלחה אמיתית", והפער **גדל עם הגודל** (~28 נק' אחוז לכל ×10 בקוד — SpecBench). reward hacking מפורש נצפה ב-Claude Code ו-Codex (EvilGenie). הגנות: **held-out tests**, זיהוי עריכת קבצי-טסט, ו-**LLM judge** (עובד מצוין על מקרים חד-משמעיים). |
| ⭐9 | **Guardrails ברמת התשתית** | הכי אפקטיבי — לא prompt. **Permissions** (deny→ask→allow, deny גובר תמיד), **hooks** שחוסמים פקודות מסוכנות (exit code 2; היה מקרה אמיתי של `rm -rf ~/`). אם אין tool — ה-agent לא יכול לבצע את הפעולה. |

### שכבה 3 — בידוד ומקביליות
| # | עמוד | פירוט |
|---|------|--------|
| 4 | **Sandbox** (שלך) | Claude Code: Bash sandbox מובנה (Seatbelt/bubblewrap) — אבל **מכסה רק Bash**. ל-file tools/MCP/hooks צריך `@anthropic-ai/sandbox-runtime`, dev container, או VM. `--dangerously-skip-permissions` רק בתוך בידוד עם **default-deny egress**. **Sandbox = defense-in-depth, לא גבול אבטחה** (היה CVE של עקיפת egress ~130 גרסאות). |
| ⭐10 | **תזמור agents מקבילים** | 3 primitives עם גבול החלטה ברור: **worktrees** מבודדים קבצים (`claude --worktree`), **subagents** מבודדים קונטקסט ומדווחים חזרה, **agent teams** (ניסיוני, ~פבר' 2026, Opus 4.6) — כל teammate session מלא שמתאם דרך task-list משותף עם file-locking. המלצה: 3–5 teammates, 5–6 משימות לכל אחד, **בעלים-קובץ אחד לכל teammate**. |

### שכבה 4 — Pipeline ואוטומציה (עמוד השדרה של ה"מפעל")
| # | עמוד | פירוט |
|---|------|--------|
| ⭐11 | **Spec-Driven Development** (זה עמוד 5 שלך, מתועש) | **GitHub Spec Kit** (Constitution→Specify→Plan→Tasks→Implement, MIT, 30+ agents). **AWS Kiro** (Requirements→Design→Tasks + EARS). **ה-spec, לא הקוד, הוא היחידה העמידה.** לולאת **PIV** (Plan-Implement-Verify) + Plan Mode. PRD = "executable build plan". |
| ⭐12 | **CI/CD + headless** | `claude -p` + `--output-format json` (מחזיר `total_cost_usd`) + `--allowedTools` + `--permission-mode dontAsk` + `--bare` (רפרודוקטיבי). PR של agent עובר דרך **אותם** branch protection / required checks / review כמו אנושי. |

### שכבה 5 — Observability ובקרת עלות (⭐ חסר נפוץ)
| # | עמוד | פירוט |
|---|------|--------|
| ⭐13 | **Tracing + עלות** | האנליטיקה הילידית של Claude Code חסרה אגרגציה צוותית (לוגים מקומיים). צוותים מוסיפים LLM gateway (LiteLLM/Bifrost/Cloudflare) + OTel (**GenAI semantic conventions**: `gen_ai.usage.input_tokens` וכו') לייחוס-עלות לכל user/key ולתקציבים. טלמטריה = הבסיס ל"self-improving software". |

### שכבה 6 — צוות וממשל
| # | עמוד | פירוט |
|---|------|--------|
| ⭐14 | **אימוץ מדורג + ממשל** | rollout מדורג (install→leverage→governance, 3 ספרינטים) ניצח דחיפה של חודש. `settings.json` משותף ב-repo: pre-approve פקודות בטוחות, block מסוכנות. בנו פלטפורמה פנימית (DORA: מנבא ההצלחה #1). |

---

## חלק 4 — תוכנית יישום מתועדפת

**עיקרון מנחה:** התשתית היא המכפיל. אל תרחיבו agents לפני שבניתם את הלולאה הסגורה — אחרת תגבירו את התפקוד הלקוי.

### גל 1 (שבועות 1–3) — "הרצפה": אמינות לפני מהירות
1. **`CLAUDE.md` רזה** (<200 שורות) + `settings.json` משותף ב-repo (allow/deny לפקודות).
2. **לולאה סגורה בסיסית:** lint + type-check + test כ-PostToolUse hooks; CI שמריץ את אותם שערים. *זה ההחזר הכי גבוה — בלי זה, הכל שאר חסר ערך.*
3. **Sandbox/בידוד:** dev container עם default-deny egress; הגדירו את הגבול לפני שנותנים autonomy.

### גל 2 (שבועות 4–8) — "הקווי ייצור": רב-פעמיות
4. **ספריית skills** משותפת + subagents מתמחים (Explore/Plan/בודק-טסטים). התחילו מ-3–5 skills למשימות הנפוצות שלכם.
5. **Spec-Driven Development:** אמצו Spec Kit (או workflow PRD→Plan→Tasks משלכם). *זה עמוד 5 שלך, מתועש.*
6. **MCP** ל-2–3 המערכות המרכזיות (issue tracker, DB, observability).

### גל 3 (שבועות 9–12) — "בקרת איכות": שתוכלו לסמוך על "ירוק"
7. **Eval suite פרטי** על הקוד שלכם + הגנות reward-hacking (held-out tests, זיהוי עריכת טסטים, LLM judge על ה-diff).
8. **Observability:** OTel + LLM gateway לייחוס-עלות ותקציבים.
9. **תשתית review** (כי זה צוואר הבקבוק): code-review אוטומטי על כל PR.

### גל 4 (שוטף) — "שורת הייצור האוטומטית"
10. **Headless ב-CI** (`claude -p`) למשימות מוגדרות-היטב.
11. **Agents מקבילים** (worktrees → agent teams) — רק אחרי שהשערים אמינים.
12. **לולאת שיפור:** טלמטריה → איתור skills/specs חלשים → שיפור (ה-ZTE של IndyDevDan).

---

## חלק 5 — מלכודות נפוצות (להימנע!)

1. **"10x אינדיבידואלי" ≠ "10x ארגוני"** — בלי האצת review/test/deploy, Amdahl בולע את הרווח.
2. **"טסטים עוברים" כ-proxy דולף** — reward hacking גדל עם הגודל; חובה held-out + judge.
3. **Sandbox כ"גבול אבטחה"** — הוא defense-in-depth בלבד (CVE-ים אמיתיים).
4. **Guardrails ברמת ה-prompt** — שבירים; אכיפה אמיתית = permissions/hooks/חוסר-tool.
5. **Worktree sprawl** — 6 agents × 5GB = 30GB+; נהלו את זה.
6. **קונטקסט מנופח** — "context rot" פוגע בביצועים; רלוונטיות > גודל.
7. **דחיפת-חודש אגרסיבית** — rollout מדורג מנצח.
8. **הסתמכות על benchmarks ציבוריים** — מזוהמים; בנו evals פרטיים.

---

## מקורות מרכזיים

**IndyDevDan:** [agenticengineer.com](https://agenticengineer.com/) · [github.com/disler/claude-code-hooks-mastery](https://github.com/disler/claude-code-hooks-mastery) · [the-library](https://github.com/disler/the-library) · [single-file-agents](https://github.com/disler/single-file-agents) · [Addy Osmani — The Factory Model](https://addyosmani.com/blog/factory-model/)

**נתונים/מציאות:** [METR RCT (arXiv 2507.09089)](https://arxiv.org/abs/2507.09089) · [DORA 2025 (Google Cloud)](https://cloud.google.com/blog/products/ai-machine-learning/announcing-the-2025-dora-report) · [Faros — DORA takeaways](https://www.faros.ai/blog/key-takeaways-from-the-dora-report-2025) · [GitClear 2025](https://www.gitclear.com/ai_assistant_code_quality_2025_research) · [Amdahl critique](https://www.augmentcode.com/guides/why-ai-doesnt-create-10x-engineering-organizations) · [review bottleneck](https://blog.logrocket.com/ai-coding-tools-shift-bottleneck-to-review/)

**Claude Code primitives:** [memory/CLAUDE.md](https://code.claude.com/docs/en/memory) · [sub-agents](https://code.claude.com/docs/en/sub-agents) · [hooks](https://code.claude.com/docs/en/hooks) · [skills](https://code.claude.com/docs/en/skills) · [mcp](https://code.claude.com/docs/en/mcp) · [permissions](https://code.claude.com/docs/en/permissions) · [headless](https://code.claude.com/docs/en/headless) · [worktrees](https://code.claude.com/docs/en/worktrees) · [agent-teams](https://code.claude.com/docs/en/agent-teams) · [sandbox-environments](https://code.claude.com/docs/en/sandbox-environments)

**Spec-driven / context engineering:** [GitHub Spec Kit](https://github.com/github/spec-kit) · [Anthropic — Effective Context Engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) · [Anthropic — Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) · [Kiro docs](https://kiro.dev/docs/specs/)

**QC / evals / reward hacking:** [Anthropic — Writing Tools for Agents](https://www.anthropic.com/engineering/writing-tools-for-agents) · [EvilGenie (arXiv 2511.21654)](https://arxiv.org/abs/2511.21654) · [SWE-Bench Pro (arXiv 2509.16941)](https://arxiv.org/pdf/2509.16941) · [Anthropic teams use Claude Code](https://claude.com/blog/how-anthropic-teams-use-claude-code)

---

### הערת אמינות מתודולוגית
WebFetch נחסם (403) על חלק מהדומיינים במהלך המחקר, כך שחלק מהטענות נשענות על תקצירי-חיפוש (לא טקסט-מלא). שלוש הטענות הכי קריטיות (METR −19%, ניסוח IndyDevDan, context-engineering של Anthropic) **אומתו ישירות**. מספרים ספציפיים מ-arXiv (SpecBench 28pp, SWE-Bench Pro 23%) כדאי לאמת מול המאמר לפני ציטוט פומבי.
