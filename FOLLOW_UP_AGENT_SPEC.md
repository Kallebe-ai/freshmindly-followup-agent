# Freshmindly Follow-Up Agent — Technical Spec

**File:** freshmindly_followup.py
**Depends on:** config.py, drafts.json
**Outputs:** Updated drafts.json, follow_up_log.txt

---

## How It Works

Run this script once per day (manually or via cron). It:

1. Loads drafts.json
2. Finds all leads where follow_up_status == "active" and next_follow_up <= today
3. Picks the correct email template based on follow_up_sequence number
4. Personalizes and sends via Gmail SMTP
5. Updates the lead record with new sequence number, last contacted date, and next follow-up date
6. Logs every send to follow_up_log.txt

---

## Script Structure

```python
"""
Freshmindly Follow-Up Agent
---------------------------
Sends scheduled follow-up emails to active leads.
Run once daily — manually or via cron.

Usage:
    python3 freshmindly_followup.py

Requirements:
    - drafts.json must exist (run freshmindly_sender.py first)
    - config.py must have GMAIL_ADDRESS and GMAIL_APP_PASSWORD set
"""

import json
import smtplib
import time
import logging
from datetime import date, timedelta
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from config import GMAIL_ADDRESS, GMAIL_APP_PASSWORD

# Config
FOLLOW_UP_DELAY_DAYS = 4          # days between follow-ups
LOG_FILE = "follow_up_log.txt"
DRAFTS_FILE = "drafts.json"
SEND_DELAY_SECONDS = 3            # pause between sends to avoid spam flags
UNSUBSCRIBE_KEYWORDS = [
    "unsubscribe", "remove me", "stop emailing", "not interested",
    "please remove", "take me off", "opt out"
]
```

---

## Data Schema Extension

When freshmindly_sender.py marks a lead as sent, the follow-up agent adds these fields on first run:

```python
def init_follow_up_fields(lead):
    """Add follow-up tracking fields to a newly-sent lead."""
    today = date.today().isoformat()
    lead.setdefault("follow_up_sequence", 0)        # 0 = initial sent, 1 = first follow-up
    lead.setdefault("last_contacted", today)
    lead.setdefault("next_follow_up", (date.today() + timedelta(days=FOLLOW_UP_DELAY_DAYS)).isoformat())
    lead.setdefault("follow_up_status", "active")   # active | replied | unsubscribed | converted | closed
    lead.setdefault("unsubscribed", False)
    lead.setdefault("replied", False)
    lead.setdefault("converted", False)
    lead.setdefault("follow_up_history", [])
    return lead
```

---

## Email Templates

```python
def get_follow_up_email(lead, sequence_num):
    """
    Returns (subject, body) tuple for the given sequence number.
    sequence_num 1 = first follow-up, 2 = second, etc.
    """
    name = lead.get("contact_name", "there")
    first_name = name.split()[0] if name != "there" else "there"
    business = lead.get("business_name", "your business")
    town = lead.get("city", "your area")

    if sequence_num == 1:
        subject = f"Re: Keep Your Office Spotless — Freshmindly"
        body = f"""Hi {first_name},

Just wanted to bump this up in case it got buried. I reached out last week about commercial cleaning for {business}.

Happy to keep it simple — even a quick 10-minute call would help me understand if we'd be a good fit.

Worth a chat?

— Kaleb
Freshmindly Commercial Cleaning
krdasilvafreshmindly@gmail.com"""

    elif sequence_num == 2:
        subject = f"Quick question about {business}'s office"
        body = f"""Hi {first_name},

I know you're busy, so I'll keep this quick — a study by Harvard Business Review found that clean workspaces can reduce employee sick days by up to 80%.

How often does {business} currently get cleaned? I might be able to save you money while doing it more frequently.

— Kaleb
Freshmindly Commercial Cleaning
krdasilvafreshmindly@gmail.com"""

    elif sequence_num == 3:
        subject = f"A couple spots opening up in {town}"
        body = f"""Hi {first_name},

I've been filling up my schedule in {town} and have a couple of weekly slots left for new clients this month.

I'd love to give {business} a free walkthrough quote — no obligation, just 15 minutes on-site. Would that work for you?

— Kaleb
Freshmindly Commercial Cleaning
krdasilvafreshmindly@gmail.com"""

    else:
        # sequence 4+ — direct and final tone
        subject = f"Last note from Freshmindly"
        body = f"""Hi {first_name},

I don't want to keep cluttering your inbox — this will be my last follow-up for now.

If the timing ever makes sense for {business}, I'd genuinely love to earn your business. You can reach me anytime at krdasilvafreshmindly@gmail.com.

Wishing you the best either way.

— Kaleb
Freshmindly Commercial Cleaning

P.S. Reply "unsubscribe" at any time and I'll remove you from my list immediately."""

    return subject, body
```

---

## Core Send Loop

```python
def run_follow_ups():
    today = date.today()

    # Load drafts
    try:
        with open(DRAFTS_FILE) as f:
            drafts = json.load(f)
    except FileNotFoundError:
        print("drafts.json not found. Run freshmindly_sender.py first.")
        return

    # Setup logging
    logging.basicConfig(
        filename=LOG_FILE,
        level=logging.INFO,
        format="%(asctime)s — %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S"
    )

    # Find eligible leads
    eligible = []
    for lead in drafts:
        if lead.get("status") != "sent":
            continue
        lead = init_follow_up_fields(lead)
        if lead["follow_up_status"] != "active":
            continue
        next_date = date.fromisoformat(lead["next_follow_up"])
        if next_date <= today:
            eligible.append(lead)

    if not eligible:
        print("No follow-ups due today.")
        return

    print(f"Sending {len(eligible)} follow-up(s)...")

    # Connect to Gmail
    server = smtplib.SMTP_SSL("smtp.gmail.com", 465)
    server.login(GMAIL_ADDRESS, GMAIL_APP_PASSWORD)

    sent_count = 0
    failed = []

    for lead in eligible:
        to_email = lead.get("to_email")
        if not to_email:
            continue

        seq = lead["follow_up_sequence"] + 1
        subject, body = get_follow_up_email(lead, seq)

        msg = MIMEMultipart()
        msg["From"] = GMAIL_ADDRESS
        msg["To"] = to_email
        msg["Subject"] = subject
        msg.attach(MIMEText(body, "plain"))

        try:
            server.sendmail(GMAIL_ADDRESS, to_email, msg.as_string())

            # Update lead record
            lead["follow_up_sequence"] = seq
            lead["last_contacted"] = today.isoformat()
            lead["next_follow_up"] = (today + timedelta(days=FOLLOW_UP_DELAY_DAYS)).isoformat()
            lead["follow_up_history"].append({
                "sequence": seq,
                "sent_at": today.isoformat(),
                "subject": subject,
                "status": "sent"
            })

            sent_count += 1
            log_msg = f"Sent follow-up #{seq} to {lead['business_name']} ({to_email})"
            print(f"  {log_msg}")
            logging.info(log_msg)
            time.sleep(SEND_DELAY_SECONDS)

        except Exception as e:
            failed.append(lead["business_name"])
            logging.error(f"Failed: {lead['business_name']} — {e}")
            print(f"  Failed: {lead['business_name']} — {e}")

    server.quit()

    # Save updated drafts
    with open(DRAFTS_FILE, "w") as f:
        json.dump(drafts, f, indent=2)

    print(f"Done! Sent: {sent_count} | Failed: {len(failed)}")


if __name__ == "__main__":
    run_follow_ups()
```

---

## Running on a Schedule

**Option A — Run manually each morning:**
```bash
cd "/Users/hbfmac/Freshmindly Prospecting Agent"
python3 freshmindly_followup.py
```

**Option B — macOS cron (runs automatically at 9 AM daily):**
```bash
crontab -e
# Add this line:
0 9 * * * cd "/Users/hbfmac/Freshmindly Prospecting Agent" && python3 freshmindly_followup.py >> follow_up_log.txt 2>&1
```

**Option C — Claude Scheduled Task:**
Ask Claude to schedule freshmindly_followup.py to run every morning via the Scheduled Tasks system.

---

## File Summary

| File | Role |
|---|---|
| freshmindly_followup.py | Main follow-up agent script |
| drafts.json | Lead data store (extended with follow-up fields) |
| follow_up_log.txt | Log of every follow-up sent / failed |
| config.py | Gmail credentials |

---

## Build Order

1. Copy this spec's code into freshmindly_followup.py
2. Run freshmindly_sender.py to populate drafts.json with sent leads
3. Run python3 freshmindly_followup.py to process the first batch of follow-ups
4. Check follow_up_log.txt to verify sends
5. Schedule daily via cron or Claude Scheduled Tasks

---

*Freshmindly Commercial Cleaning · Chelmsford, MA · krdasilvafreshmindly@gmail.com*
