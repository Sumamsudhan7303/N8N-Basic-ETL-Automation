What I built:

An n8n workflow that automatically cleans 50 messy student enrollment records using AI, then sorts them into 4 separate files based on two conditions — city and email validity.

The flow, step by step:

CSV Upload — trigger, starts when the file is uploaded
Extract from File — reads the CSV, turns it into 50 individual rows
Basic LLM Chain (OpenAI) — sends each row to AI with cleaning rules: fix name casing, flag invalid emails, flag missing phone numbers, standardize course names, convert Fee_Paid to true/false, standardize city casing, fix date format
Loop Over Items — processes each of the 50 rows one at a time
Code in JavaScript — parses the AI's response (which comes back as text) into real JSON, and independently checks if the email is actually valid using a regex — this way validation isn't left purely to the AI's judgment
If node (City) — checks if City = Chennai → splits into Chennai vs Other cities
If1 and If2 (Email) — each branch checks Email_Valid = true → splits into valid vs invalid emails

Final result — 4 output files:

chennai_valid_email
chennai_invalid_email
other_valid_email
other_invalid_email

That's the whole thing — raw messy CSV in, AI cleans it, code double-checks the important bits, and it comes out the other end pre-sorted into 4 clean, ready-to-use files.
