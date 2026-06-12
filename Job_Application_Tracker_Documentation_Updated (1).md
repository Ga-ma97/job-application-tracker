# Job Application Tracking Workflow

**By Garima M Kadwe**

## What This Does
Every time a job application confirmation email lands in Gmail, this workflow automatically:
* Extracts the company name, job role, date, sender, and subject
* Logs everything into a Google Sheet
* Sends an instant Telegram alert with all details
* Uses Google Gemini AI to score how well the opportunity matches a software intern profile (1–10)

No manual tracking. No missed applications. It just runs — silently, in the background, 24/7.

## Tools Used
| Tool | Purpose |
| :--- | :--- |
| n8n Cloud | Workflow automation engine |
| Gmail | Email trigger source |
| Google Sheets | Application log |
| Telegram Bot | Real-time alerts |
| Google Gemini AI | Match score generation |

## Workflow Architecture
Gmail Trigger (polls every minute) → Code Node (extracts + deduplicates) → Basic LLM Chain — Google Gemini (AI scoring) → Google Sheets (appends new row) → Telegram Bot (sends formatted alert)

## The AI Feature
Beyond the base task requirements, I added a Google Gemini AI scoring node that reads the email subject and company name, then rates the opportunity 1–10 for a software/tech intern role. This turns the tracker into a prioritisation tool — you can glance at the Telegram alert and immediately know whether to follow up or not.

## Google Sheet Structure
Row 1 is frozen so headers are always visible.

| Col | Header | Example |
| :--- | :--- | :--- |
| A | Company Name | Amazon |
| B | Job Role | Software Engineer Intern |
| C | Application Date | 2026-06-09 |
| D | Email Subject | Your application for... |
| E | Sender Email | no-reply@amazon.com |
| F | Message ID | 19eac03163891 |
| G | Alert Sent | yes |
| H | Match Score | 8 |

## Telegram Alert Format
> 🎯 **New Job Application Logged!**
>
> 🏢 **Company:** Amazon
> 
> 💼 **Role:** Software Engineer Intern
> 
> 📅 **Date:** 2026-06-09
> 
> 📧 **From:** no-reply@amazon.com
> 
> 📨 **Subject:** Your application for Software Engineer Intern at Amazon
> 
> ⭐ **Match Score:** 8/10
> 
> ✅ Saved to Google Sheets!

## How to Set This Up

### 1. Accounts needed
* n8n Cloud (free trial at n8n.io)
* Gmail account
* Google Sheets (same Google account)
* Telegram — create a bot via @BotFather, get your Chat ID from @userinfobot
* Google AI Studio API key (aistudio.google.com) — free tier

### 2. Import the workflow
In n8n, click "..." → Import from file. Upload the provided .json file.

<h3>3. Connect credentials</h3>
* **Gmail:** connect via OAuth (Sign in with Google)
* **Google Sheets:** connect via OAuth (same Google account)
* **Telegram:** paste your Bot Token from @BotFather
* **Gemini:** paste your API key from Google AI Studio

### 4. Set up Google Sheet
* Create a sheet named "Job Application Tracker"
* Add headers in Row 1: Company Name, Job Role, Application Date, Email Subject, Sender Email, Message ID, Alert Sent, Match Score
* Freeze Row 1 (View → Freeze → 1 row)

### 5. Publish
Click "Publish" in n8n. The workflow now runs automatically every minute.

## Duplicate Prevention
Duplicate prevention is handled directly in the JavaScript code node. Each incoming email carries a unique Message ID assigned by Gmail. Before processing, the code checks whether this ID has already been seen in the current execution batch — if it has, it returns an empty array, stopping the workflow silently. For cross-session duplicate prevention (across different workflow executions), the Message ID is stored in Column F of the Google Sheet, which can be used in future iterations to build a lookup check against historical data.

## Challenges Faced & How I Solved Them
* **Gmail field naming inconsistency:** Gmail passed the sender field as *From* (capital F) in some cases and *from* (lowercase) in others. The workflow crashed silently on real emails while working fine on test ones. Fixed by checking both variations: `email.From || email.from`.
* **Telegram "chat not found" error:** The bot token was correct but the Chat ID kept returning errors. Root cause: I hadn't sent a message to the bot first, so Telegram hadn't registered the chat. Sending `/start` to the bot resolved it. Also created a second bot (JobTracker2026v2bot) to start fresh.
* **Column mapping breaking after JSON import:** When I reimported the workflow JSON after testing, the Google Sheets node lost its column references and showed "undefined" for all fields. Fixed by explicitly referencing the upstream node by name: `$('Code in JavaScript1').item.json.company` instead of `$json.company`.
* **AI quota and temporary Gemini service errors:** While testing the Google Gemini integration, the workflow occasionally encountered HTTP 503 service unavailable errors from the Gemini endpoint. To improve reliability, I implemented an automatic retry mechanism that re-runs the Gemini node up to **three times** whenever a 503 error occurs. Additionally, the node was configured with **Continue On Error** enabled, ensuring that the remainder of the workflow (Google Sheets logging and Telegram notifications) continues executing even if the AI scoring step temporarily fails. This prevented workflow interruptions while maintaining consistent application tracking and alert delivery.
* **Role and company extraction limitations from subject lines:** Because certain confirmation emails featured generic subject lines that omitted essential corporate or position details, the system initially logged these parameters as "Not Specified." To resolve this limitation and establish a more robust parsing architecture, I enhanced the underlying extraction code to programmatically scan and analyze both the email subject line and the core message body sequentially.
