# Student Enrollment Data Cleaner (n8n Workflow)

An n8n automation that takes messy student enrollment rows, cleans them using an AI node, and standardizes the output into a consistent, rule-validated JSON format — ready for reporting, storage, or downstream use.

## What it does

Raw enrollment data is often inconsistent — mixed casing, invalid emails, incomplete phone numbers, non-standard course names, and inconsistent date formats. This workflow fixes all of that automatically, applying a fixed set of business rules to every row.

**Input** (raw, messy):
```json
{
  "Student_ID": "STU1001",
  "Name": "manoj k",
  "Email": "manojk@gmail",
  "Phone": "9876543210",
  "Course": "python",
  "Fee_Paid": "yes",
  "City": "Chennai",
  "Enrolled_Date": "23-05-2025"
}
```

**Output** (cleaned, standardized):
```json
{
  "Student_ID": "STU1001",
  "Name": "Manoj K",
  "Email": "INVALID_EMAIL",
  "Phone": "9876543210",
  "Course": "Python",
  "Fee_Paid": true,
  "City": "Chennai",
  "Enrolled_Date": "2025-05-23",
  "Email_Valid": false
}
```

## Cleaning Rules Applied

| Field | Rule |
|---|---|
| `Name` | Converted to Title Case |
| `Email` | If invalid format, replaced with `INVALID_EMAIL`; `Email_Valid` flag set accordingly |
| `Phone` | If fewer than 10 digits after stripping non-numeric characters, set to `MISSING` |
| `Course` | Must be one of: `Python`, `Machine Learning`, `Data Science`; anything else becomes `INVALID_COURSE` |
| `Fee_Paid` | Normalized from `yes/YES/true` variants into a proper boolean |
| `City` | Converted to Title Case; empty values become `UNKNOWN` |
| `Enrolled_Date` | Converted from `DD-MM-YYYY` to `YYYY-MM-DD` |
| `Email_Valid` | Computed via regex — independent, reliable check (not left to AI judgment) |

## Workflow Structure

```
[Extract from File]  →  [AI Node (cleaning prompt)]  →  [Code Node (parse + validate)]
      50 raw rows          LLM cleans each row              Final structured JSON
```

1. **Extract from File** — loads the raw enrollment dataset (50 student rows)
2. **AI Node** — runs each row through an LLM with a strict cleaning prompt (see below), returning cleaned data as a JSON string
3. **Code Node** — parses the AI's string output into real JSON and recalculates `Email_Valid` deterministically using regex (AI output isn't trusted for this — rule-based checks are more reliable than relying on model judgment)

## The AI Prompt Used

```
Clean this student enrollment row and return ONLY a JSON object.

Row data: {{ JSON.stringify($json) }}

Rules:
- Name: Title Case (Manoj K)
- Email: if invalid set INVALID_EMAIL
- Phone: if less than 10 digits set MISSING
- Course: only Python or Machine Learning or Data Science
- Fee_Paid: yes/YES = true, no/NO = false
- City: Title Case, empty = UNKNOWN
- Enrolled_Date: convert to YYYY-MM-DD

Return ONLY raw JSON. No explanation. No markdown.
```

## The Code Node Logic

Key implementation detail: the AI node's response arrives wrapped as `item.json.text` — a **string** containing JSON, not an object. The code parses this first before applying any further logic.

```javascript
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
```

## Key Design Decisions

- **Email validity is computed in code, not trusted from the AI.** LLM judgment on formatting rules can vary row to row; a regex check is 100% consistent across all records.
- **Errors don't crash the whole batch.** If one row's AI output fails to parse, it's flagged with `parse_error: true` and the rest of the 50 rows still process normally.
- **AI handles the fuzzy part (interpreting messy real-world text), code handles the strict part (validation, formatting).** This hybrid is more reliable than asking the AI to do everything itself.

## Debugging Note

If Code node output doesn't match expectations, the fastest fix is always to inspect the raw data shape before changing logic:

```javascript
// Temporary debug — reveals the exact incoming data structure
return [{ json: $input.all()[0] }];
```

This surfaces any unexpected nesting (e.g., data wrapped inside `.text`, `.output`, or similar fields) before you write parsing logic around assumptions.

## Requirements

- n8n (self-hosted or cloud)
- An AI/LLM node (OpenAI, Claude, or similar) connected with a valid API key
- Input dataset of student enrollment records (JSON or file source)

## Setup

1. Import the workflow JSON into your n8n instance
2. Connect your AI node credentials (API key)
3. Point the "Extract from File" node to your dataset
4. Execute the workflow

---

Built while learning n8n automation and AI-assisted data pipelines as part of the Social Eagle AI course.
