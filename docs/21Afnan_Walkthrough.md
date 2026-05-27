# Comprehensive Walkthrough: Multi-Role Hiring Automation System
**Developer:** Afnan Shoukat
**Tools Used:** n8n, Python, Google Workspace, LLMs (Gemini/Groq)

---

## 🌟 Executive Summary
This system is an end-to-end recruitment engine designed to eliminate manual resume screening. It automatically captures applications for multiple roles, standardizes the data using Python, evaluates resumes using Role-Specific AI Rubrics, and manages the communication lifecycle with candidates.

---

## 🖼️ High-Level Workflow
![n8n Hiring Automation Workflow](../assets/workflow_logic.png)
*(Note: Please ensure the image is saved as `workflow_logic.png` in the `assets/` folder for it to appear in the document.)*

---

## 🏗️ Technical Architecture & Roadmap

The system works like a "hub-and-spoke" model where **n8n** acts as the central controller connecting various technical layers.

1.  **Ingestion Layer**: Takes applications from multiple Google Sheets (SWE and BDM).
2.  **Standardization Layer**: A Python script cleans the data so it all follows the same format.
3.  **Persistence Layer**: A "Master Sheet" acts as the database for all candidates.
4.  **Intelligence Layer**: Different AI personalities (Rubrics) are used to score candidates.
5.  **Synchronization Layer**: The system syncs back to original sheets to mark tasks as "Done."

---

## 🔄 The Step-by-Step Workflow

### 1. Multi-Source Ingestion
The system is triggered on a schedule (e.g., every hour) to batch-process entries. It checks two specific sources:
*   **Software Engineers (SWE)**
*   **Business Development Managers (BDM)**
Each incoming row is tagged with its role type before being merged into the processing stream.

### 2. Intelligent Data Cleaning (Python Layer)
I implemented a robust Python script to handle real-world data issues:
*   **Data Validation:** It cleans extra spaces, fixes missing info, and handles null values.
*   **Unique Candidate IDs:** It creates an ID by hashing the email and role. This prevents "Application Spam" (someone applying twice for the same job).
*   **Resume Extraction:** It converts shared Google Drive links into "Direct Download IDs" so the system can pull the PDF programmatically.

### 3. AI Resume Evaluation (The Brain)
This is the most critical technical phase:
*   **PDF Extraction:** The system downloads the resume directly from Google Drive and extracts the raw text.
*   **Role-Specific Scoring:** Instead of a generic check, I programmed the AI with different "Hiring Personalities":
    *   **For Engineers:** It evaluates tech stack depth, system design experience, and coding impact.
    *   **For Sales Managers:** It evaluates revenue targets, client relationship management, and sales growth.
*   **JSON Data Model:** The AI returns a structured score (0-100), classification (Strong/Average/Weak), and specific strengths/concerns.

### 4. Results & Communication Strategy
*   **Central Storage:** All information and AI insights are saved to the **Master Sheet**.
*   **Feedback Loop:** The system returns to the *original* form sheets and marks processed candidates as "Done" to ensure they aren't screened twice.
*   **Smart Emailing:** To avoid confusion, I **removed the person from the calendar invite** and used a dedicated **Custom Email node**. This ensures the candidate receives exactly one professional invitation formatted properly.

---

## ⚠️ Known Assumptions & Limitations

To maintain reliability, the system follows these specific rules:

1.  **PDF Format Only:** The text extraction is optimized for `.pdf` resumes. Photos or Word docs are flagged as "Screening Errors" for manual check.
2.  **Google Drive Access:** The resume link must be "viewable" so the API can reach the file.
3.  **Link Format:** The script is designed to parse Google Drive URLs. Files on external clouds (Dropbox, etc.) need an update to be supported.
4.  **AI Tone:** While the scoring (0-100) is consistent, the generated summaries might vary slightly in wording as is common with LLMs.

---

## 🚀 Future Roadmap
*   **LLM-Personalized Outreach:** I plan to upgrade the static templates to "AI-written" emails that mention specific highlights from a candidate's resume.
*   **Multi-Format Support:** Adding support for OCR to read image-based resumes and Word documents.

---

## 🛠️ Reliability & Error Handling
*   **Safe Skips:** If a mandatory field (like an email) is missing, the system logs the error and moves to the next person instead of crashing.
*   **Data Validation:** A final script double-checks the AI's answer format to ensure it "fits" the Master Sheet perfectly before saving.

**Summary**: This system transforms a giant pile of resumes into a clean, ranked database, allowing recruiters to focus only on the top 1% of talent.





