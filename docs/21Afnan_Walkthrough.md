# Walkthrough: Multi-Role Hiring Automation System
**By:** Afnan Shoukat
**Role:** n8n Automation Developer

---

## 1. How the System Works

This system is built to handle the entire hiring process automatically. It is split into two main parts to keep everything organized and reliable.

### Part 1: Finding & Scoring Candidates
- **Reading Data:** The system automatically checks for new applications from two different Google Sheets (one for **Software Engineers** and one for **BDMs**).
- **Cleaning Data:** Since different forms ask different questions, a Python script cleans and standardizes the info into one single format. 
- **No Duplicates:** The system gives every candidate a unique ID based on their email. If someone applies twice, it just updates their info instead of creating a new row.
- **AI Specialist:** It "reads" the resume PDF using AI. Most importantly, it uses different rules for each role—a developer is judged on tech skills, while a BDM is judged on sales targets.
- **Saving Results:** All the AI scores and summaries are saved into one **Master Sheet**.

### Part 2: Sending Emails & Scheduling
- **Batch Processing:** Every few hours, the system checks the Master Sheet for candidates who haven't been contacted yet.
- **Automatic Emails:**
    - **Top Candidates (Strong):** It automatically creates a **Google Calendar** meeting and sends them an invitation email with the link.
    - **Medium Match (Average):** It alerts the **Hiring Manager** to take a manual look.
    - **Not a Match (Weak):** it sends a professional, polite rejection email automatically.

---

## 2. Key Decisions

- **Two Separate Steps:** I separated "Scoring" from "Emailing." This ensures that if the email server has a temporary issue, the AI screening doesn't stop working.
- **Smart IDs:** By using a unique ID (email + role), the system never gets confused by duplicate applications.
- **Role-Aware AI:** Instead of a generic check, the AI knows exactly what to look for in a developer vs. a salesperson.

---

## 3. Assumptions & Limits

- **Reading PDFs:** The system works best with standard PDF resumes. If a candidate uploads a low-quality photo of a resume, the AI might find it harder to read.
- **Calendar Timing:** Currently, it schedules interviews for 3 days in the future. In a real scenario, this would be connected to a live calendar like Calendly.

---

## 4. Handling Problems

- **Missing Info:** If a candidate forgets to include their name or resume link, the system gracefully skips them and logs a note instead of crashing.
- **AI Safety:** Sometimes AI output can be messy. I added a "safety" script that double-checks the AI's results and fixes any formatting errors before saving them to the sheet.
- **Graceful Stop:** If there are no new applicants, the system simply stops and waits for the next scheduled run without doing unnecessary work.

