Student Enrollment ETL Pipeline (n8n)

CAIE Course — Assignment 7: AI Automation ETL Pipeline Challenge

An n8n workflow that extracts raw, messy student enrollment data from CSV, cleans it using AI, validates it with rule-based logic, and routes each record into one of four output files based on two filtering conditions.

Problem

Social Eagle AI Academy receives weekly student enrollment data via Google Forms. The raw data is inconsistent — mixed casing, invalid emails, missing phone digits, inconsistent date formats. This pipeline automates cleaning and sorting it into ready-to-use files.

Pipeline Overview
CSV Upload → Switch → Extract from File → Basic LLM Chain → Loop Over Items
                                             (OpenAI Chat Model)      ↓
                                                            Code in JavaScript
                                                                      ↓
                                                          Loop Over Items (until done)
                                                                      ↓
                                                        If — City = Chennai?
                                          ┌───────────true───────────┴───────────false──────────┐
                                          ↓                                                       ↓
                                  If1 — Email valid?                                    If2 — Email valid?
                              ┌─────true────┴────false────┐                      ┌─────true────┴────false────┐
                              ↓                            ↓                      ↓                            ↓
                    chennai_valid_email          chennai_invalid_email    other_valid_email          other_invalid_email
Output Files
File	Condition
chennai_valid_email	City = Chennai AND Email valid
chennai_invalid_email	City = Chennai AND Email invalid
other_valid_email	City = Other AND Email valid
other_invalid_email	City = Other AND Email invalid
Nodes Used
Node	Purpose
CSV Upload	Trigger — starts the workflow when a file is uploaded
Switch	Routes the uploaded file
Extract from File	Reads the CSV into 50 structured rows
Basic LLM Chain + OpenAI Chat Model	AI cleans each row per the prompt rules below
Loop Over Items	Processes rows one at a time through the Code node until all 50 are done
Code in JavaScript	Parses the AI's raw text output into real JSON, applies deterministic validation (especially Email_Valid)
If	Splits rows by City = Chennai
If1	Splits Chennai rows by Email_Valid
If2	Splits Other-city rows by Email_Valid
Convert to File (×4)	Exports each of the 4 final branches as a separate file
AI Cleaning Prompt (Basic LLM Chain)
Clean this student enrollment row and return ONLY a JSON object.

Row data: {{ JSON.stringify($json) }}

Rules:
- Name: Title Case
- Email: if invalid (no @ or no .com/.in) set INVALID_EMAIL
- Phone: if less than 10 digits or empty set MISSING
- Course: only Python or Machine Learning or Data Science
- Fee_Paid: yes/YES = true, no/NO = false
- City: Title Case, empty = UNKNOWN
- Enrolled_Date: YYYY-MM-DD format

Return ONLY raw JSON. No explanation. No markdown.
Code Node Logic

The AI node's response arrives as item.json.text — a string containing JSON, not a parsed object. This must be parsed before any transformation logic runs.

javascript
// Mode: Run Once for All Items

function toTitleCase(str) {
  if (!str) return '';
  return str.replace(/\w\S*/g, txt =>
    txt.charAt(0).toUpperCase() + txt.substr(1).toLowerCase()
  );
}

function formatDate(dateStr) {
  if (!dateStr) return '';
  const parts = dateStr.split('-');
  if (parts.length === 3 && parts[0].length === 2) {
    return `${parts[2]}-${parts[1]}-${parts[0]}`;
  }
  return dateStr;
}

const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
const allowedCourses = ['python', 'machine learning', 'data science'];

return $input.all().map(item => {
  let data;

  try {
    data = JSON.parse(item.json.text);
  } catch (error) {
    return {
      json: {
        parse_error: true,
        raw_text: item.json.text
      }
    };
  }

  const emailValid = emailRegex.test(data.Email);
  const email = emailValid ? data.Email : 'INVALID_EMAIL';

  const digitsOnly = (data.Phone || '').replace(/\D/g, '');
  const phone = digitsOnly.length >= 10 ? digitsOnly : 'MISSING';

  const courseLower = (data.Course || '').toLowerCase().trim();
  const course = allowedCourses.includes(courseLower)
    ? toTitleCase(courseLower)
    : 'INVALID_COURSE';

  const feePaidRaw = String(data.Fee_Paid).toLowerCase();
  const feePaid = feePaidRaw === 'yes' || feePaidRaw === 'true';

  const city = data.City ? toTitleCase(data.City) : 'UNKNOWN';

  return {
    json: {
      Student_ID: data.Student_ID || '',
      Name: toTitleCase(data.Name),
      Email: email,
      Phone: phone,
      Course: course,
      Fee_Paid: feePaid,
      City: city,
      Enrolled_Date: formatDate(data.Enrolled_Date),
      Email_Valid: emailValid
    }
  };
});
Filter Node Conditions

If (City filter):

{{ $('Code in JavaScript').item.json.City }}  is equal to  Chennai

If1 and If2 (Email filter — identical condition, different branch position):

{{ $('Code in JavaScript').item.json.Email_Valid }}  is equal to  true

(Comparison type: Boolean)

Key Design Decisions
Email_Valid is computed in code via regex, not trusted from the AI. Rule-based validation is 100% consistent across all 50 rows; LLM judgment on formatting can vary row to row.
Errors don't crash the whole batch. If one row's AI output fails to parse, it's flagged with parse_error: true while the rest continue processing normally.
AI handles the fuzzy part (interpreting messy real-world text formatting), code handles the strict part (validation, boolean logic, date/phone format enforcement). This hybrid is more reliable than asking the AI to do everything end-to-end.
Debugging Note

If a Code node's output doesn't match expectations, inspect the raw data shape before changing logic — don't assume it:

javascript
// Temporary debug — reveals the exact incoming data structure
return [{ json: $input.all()[0] }];

This is how the item.json.text nesting (a common trap with AI/LLM node outputs in n8n) was discovered during development.

Bonus Output (Optional)

A 5th output — chennai_paid_confirmed — can be added by branching off the Chennai + Valid Email path with one more IF node checking Fee_Paid is true.

Requirements
n8n (self-hosted; this project runs on a Hostinger VPS instance)
OpenAI API key (or equivalent LLM provider) connected to the Basic LLM Chain node
student_enrollment_raw.csv — 50 rows of raw student enrollment data
Setup
Import the workflow JSON into your n8n instance
Connect your OpenAI (or other LLM) credentials to the Chat Model node
Upload student_enrollment_raw.csv via the CSV Upload trigger
Click Execute Workflow
Check the 4 output files for correct routing

Built as part of the Social Eagle CAIE Course Program — AI Automation track.
