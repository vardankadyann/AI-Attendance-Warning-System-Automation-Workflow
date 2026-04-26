# AI-Attendance-Warning-System-Automation-Workfolow
# AI-Powered Attendance & Warning System

An HR automation system built with n8n that processes biometric attendance data, detects late arrivals, manages a three-strike warning system, and uses AI to generate personalized warning emails.

## Features
- 📊 Ingests daily Excel exports from biometric machines
- 🕐 Auto-detects late arrivals (check-in after 11:00 AM)
- ⚡ Three-strike warning system with escalating AI-generated emails
- 📅 Auto-schedules 5 PM HR meeting on 3rd strike
- 📋 Google Sheets storage (DailyRecords + MonthlyLateCounter)
- 🚨 Missing punch detection with HR alerts
- 📈 Real-time HR dashboard (Bonus)

## Tech Stack
- **Orchestration:** n8n
- **Storage:** Google Sheets
- **AI:** OpenAI / Groq LLM
- **Communication:** Gmail + Google Calendar API

## Setup
1. Import `n8n_workflow.json` into n8n
2. Add credentials (Google Sheets, Gmail, Google Calendar, OpenAI)
3. Set `SHEET_ID` variable to your Google Sheet ID
4. Create two tabs in Google Sheets: `DailyRecords` and `MonthlyLateCounter`
5. Activate the workflow

## Architecture
See `documentation.md` for full architecture diagram, edge case handling, and scalability plan.
