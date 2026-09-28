# TCS-hackathon

"""Scholarship Finder Bot (command-line version).

Run:  python scholarship_finder_bot.py
Needs Python 3.8+ and no extra packages.
"""

from datetime import date

# ---- Scholarship data (edit or extend) ----
SCHOLARSHIPS = [
    {"name": "Merit Excellence Award", "amount": 5000, "min_gpa": 3.5, "max_income": None,
     "levels": ["undergraduate"], "fields": None, "gender": None, "countries": None,
     "deadline": "2026-12-15"},
    {"name": "STEM Future Leaders Grant", "amount": 4000, "min_gpa": 3.0, "max_income": None,
     "levels": ["undergraduate", "postgraduate"],
     "fields": ["engineering", "computer science", "mathematics", "science"],
     "gender": None, "countries": None, "deadline": "2026-11-30"},
    {"name": "Women in Tech Scholarship", "amount": 3500, "min_gpa": 3.0, "max_income": None,
     "levels": ["undergraduate", "postgraduate"], "fields": ["computer science", "engineering"],
     "gender": "female", "countries": None, "deadline": "2027-01-20"},
    {"name": "Need-Based Education Fund", "amount": 6000, "min_gpa": 2.5, "max_income": 30000,
     "levels": ["undergraduate"], "fields": None, "gender": None, "countries": None,
     "deadline": "2026-10-31"},
    {"name": "Global Postgraduate Fellowship", "amount": 10000, "min_gpa": 3.6, "max_income": None,
     "levels": ["postgraduate"], "fields": None, "gender": None, "countries": None,
     "deadline": "2027-02-28"},
    {"name": "National Youth Scholarship", "amount": 2500, "min_gpa": 2.8, "max_income": 50000,
     "levels": ["high school", "undergraduate"], "fields": None, "gender": None,
     "countries": ["india"], "deadline": "2026-12-01"},
    {"name": "Arts & Humanities Bursary", "amount": 2000, "min_gpa": 2.5, "max_income": None,
     "levels": ["undergraduate", "postgraduate"],
     "fields": ["arts", "literature", "history", "music"],
     "gender": None, "countries": None, "deadline": "2026-11-15"},
]

# ---- Conversation steps ----
STEPS = [
    {"key": "name", "q": "Hi! I'm the Scholarship Finder Bot. What's your name?", "type": "text"},
    {"key": "level", "q": "Nice to meet you, {name}! What's your education level?",
     "type": "choice", "opts": ["high school", "undergraduate", "postgraduate"]},
    {"key": "field", "q": "What's your field of study? (pick a number or type your own)",
     "type": "choice",
     "opts": ["computer science", "engineering", "science", "mathematics", "arts", "other"],
     "allow_text": True},
    {"key": "gpa", "q": "What's your GPA on a 4.0 scale?", "type": "number", "min": 0, "max": 4},
    {"key": "income", "q": "What's your annual family income? (number only)",
     "type": "number", "min": 0},
    {"key": "gender", "q": "Gender?", "type": "choice", "opts": ["female", "male", "other"]},
    {"key": "country", "q": "Which country do you live in?", "type": "text"},
]


# ---- Eligibility + ranking ----
def check(st, s):
    """Return a list of reasons the student does NOT qualify (empty = eligible)."""
    fails = []
    if st["gpa"] < s["min_gpa"]:
        fails.append(f"needs GPA of {s['min_gpa']} or higher")
    if s["max_income"] is not None and st["income"] > s["max_income"]:
        fails.append(f"family income must be under {s['max_income']:,}")
    if st["level"] not in s["levels"]:
        fails.append("only for " + " / ".join(s["levels"]))
    if s["fields"] and st["field"] not in s["fields"]:
        fails.append("only for " + ", ".join(s["fields"]))
    if s["gender"] and st["gender"] != s["gender"]:
        fails.append(f"only for {s['gender']} applicants")
    if s["countries"] and st["country"] not in s["countries"]:
        fails.append("only for residents of " + ", ".join(s["countries"]))
    if date.fromisoformat(s["deadline"]) < date.today():
        fails.append("deadline has passed")
    return fails


def score(st, s):
    """Higher = better match (bigger award, more specific, bigger GPA margin)."""
    p = s["amount"] / 1000
    if s["fields"]:
        p += 2
    if s["gender"] or s["countries"]:
        p += 1
    return p + (st["gpa"] - s["min_gpa"])


# ---- Input helpers ----
def bot(text):
    print(f"\n🤖 {text}")


def ask_step(step, st):
    bot(step["q"].format(**st))
    if step["type"] == "choice":
        for n, opt in enumerate(step["opts"], 1):
            print(f"   {n}. {opt}")

    while True:
        raw = input("You: ").strip()
        if not raw:
            continue

        if step["type"] == "number":
            try:
                val = float(raw.replace(",", ""))
            except ValueError:
                val = None
            hi = step.get("max")
            if val is None or val < step["min"] or (hi is not None and val > hi):
                msg = "That doesn't look right. Enter a number"
                if hi is not None:
                    msg += f" between {step['min']} and {hi}"
                bot(msg + ".")
                continue
            return val

        if step["type"] == "choice":
            opts = step["opts"]
            if raw.isdigit() and 1 <= int(raw) <= len(opts):
                return opts[int(raw) - 1]
            if raw.lower() in opts:
                return raw.lower()
            if step.get("allow_text"):
                return raw.lower()
            bot("Please pick one of the listed options.")
            continue

        # plain text
        return raw.title() if step["key"] == "name" else raw.lower()


def print_card(s, note=None):
    print(f"\n   • {s['name']}")
    if note is None:
        print(f"     ${s['amount']:,}   Deadline: {s['deadline']}")
    else:
        print(f"     Missing: {note}")


def finish(st):
    ok, near = [], []
    for s in SCHOLARSHIPS:
        fails = check(st, s)
        if not fails:
            ok.append(s)
        elif len(fails) == 1 and not fails[0].startswith("deadline"):
            near.append((s, fails[0]))

    ok.sort(key=lambda s: score(st, s), reverse=True)

    if ok:
        plural = "s" if len(ok) > 1 else ""
        bot(f"Good news, {st['name']}! You qualify for {len(ok)} scholarship{plural}, "
            "best match first:")
        for s in ok[:5]:
            print_card(s)
    else:
        bot("No exact matches right now.")

    if near:
        bot("You're one requirement away from these:")
        for s, why in near:
            print_card(s, why)
    print()


def main():
    print("=" * 50)
    print("  Scholarship Finder Bot")
    print("  Answer 7 quick questions to see scholarships")
    print("  you qualify for.  (Ctrl+C to quit)")
    print("=" * 50)
    try:
        while True:
            st = {}
            for step in STEPS:
                st[step["key"]] = ask_step(step, st)
            finish(st)
            again = input("Start over? (y/n): ").strip().lower()
            if again != "y":
                break
        print("Goodbye! Good luck with your applications.")
    except (KeyboardInterrupt, EOFError):
        print("\nGoodbye!")


if __name__ == "__main__":
    main()
