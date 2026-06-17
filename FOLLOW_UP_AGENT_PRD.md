# Freshmindly Follow-Up Agent — Product Requirements Document (PRD)

**Version:** 1.0
**Date:** June 16, 2026
**Owner:** Kallebe Da Silva — Freshmindly Commercial Cleaning
**Status:** Draft

---

## 1. Overview

The Follow-Up Agent is an automated email sequencer that sends personalized follow-up emails to cold outreach leads on behalf of Freshmindly. It picks up where freshmindly_sender.py leaves off — once an initial cold email is marked sent, the Follow-Up Agent manages the entire follow-up lifecycle until the lead replies, unsubscribes, or is manually closed.

---

## 2. Problem Statement

After the initial cold email is sent, leads rarely convert on the first touch. Studies show 80% of sales require 5+ follow-ups, yet most businesses stop after 1. Without automation, Freshmindly has no way to systematically follow up with hundreds of leads.

---

## 3. Goals

- Primary: Increase reply rate from cold outreach by maintaining consistent, non-spammy follow-up contact.
- Secondary: Save Kaleb time — zero manual effort required after the initial email is sent.
- Guardrail: Never email a lead who has replied or asked to be removed.

---

## 4. Success Metrics

| Metric | Target |
|---|---|
| Follow-up open rate | > 35% |
| Reply rate (all touches combined) | > 5% |
| Unsubscribe/complaint rate | < 1% |
| Time Kaleb spends on follow-ups | 0 minutes |

---

## 5. Follow-Up Sequence Design

| Touch | Timing | Tone | Goal |
|---|---|---|---|
| Initial email | Day 0 | Personalized intro | Awareness |
| Follow-up 1 | Day 3-5 | Short & casual check-in | Re-engagement |
| Follow-up 2 | Day 3-5 after 1 | Value-add (stat/question) | Build credibility |
| Follow-up 3 | Day 3-5 after 2 | Soft urgency + social proof | Create FOMO |
| Follow-up 4+ | Day 5 after each prior | Direct, final-attempt energy | Last push |

---

## 6. Stop Conditions

1. Lead replies
2. Unsubscribe — lead replies with "unsubscribe", "remove me", "stop", "not interested"
3. Manual close — Kaleb marks the lead as closed in the dashboard
4. Hard bounce — email delivery fails permanently
5. Converted — Kaleb marks as converted (became a client)

---

## 7. Technical Requirements

| Requirement | Detail |
|---|---|
| Language | Python 3 |
| Email delivery | Gmail SMTP via App Password |
| Scheduling | Run daily via cron job or manually |
| Data store | drafts.json (extended schema) |
| Reply detection | Manual flag in dashboard (v1) / Gmail API (v2) |
| Logging | Append to follow_up_log.txt |

---

## 8. Rollout Plan

1. Initial cold email system (complete)
2. Extend drafts.json schema with follow-up fields
3. Build freshmindly_followup.py per FOLLOW_UP_AGENT_SPEC.md
4. Update dashboard to show follow-up status per lead
5. Test on 5 leads manually before full rollout
6. Schedule daily run via cron or Claude scheduled task

---

*Freshmindly Commercial Cleaning · Chelmsford, MA · krdasilvafreshmindly@gmail.com*
