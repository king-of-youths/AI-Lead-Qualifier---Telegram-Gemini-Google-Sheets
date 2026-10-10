Readme · MD
AI Lead Qualifier
A Telegram bot that qualifies sales leads automatically. A customer describes what they need, the bot asks follow-up questions for anything that is missing, and the finished lead is saved to Google Sheets.

Stack: n8n · Telegram · Google Gemini · Google Sheets

What it does
Chats with customers on Telegram in plain language.
Collects seven details: Customer Name, Phone Number, Business Type, Service Needed, Budget, Timeline, Urgency.
Understands customers who give everything in one message, and only asks for what is still missing.
Uses tap-to-answer buttons for Service Needed and Urgency. Customers can also type their answer.
Greets the customer by first name and uses emojis in its messages.
Saves each completed lead to Google Sheets with a unique Lead ID and the customer's original message.
Replies politely if the customer says "thanks" after finishing.
Design principle
The AI only extracts information. n8n controls the conversation.

Gemini does	n8n does
Reads the customer's message	Looks up the customer's session
Returns structured JSON	Merges new data with saved data
Never invents missing details	Finds the first missing field
Chooses the next question, buttons, and wording
Generates the Lead ID and writes to Google Sheets
Handles errors and sends Telegram replies
How it works
Telegram message or button tap
        |
        v
Extract Telegram Data  ->  (button tap? acknowledge it)
        |
        v
Find Session (Google Sheets)  ->  Prepare Session And Prompt
        |
        |-- thank-you after a finished lead? --> Reply To Thanks
        v
Gemini Extraction  ->  Parse And Validate JSON
        |
        v
Merge And Check Fields
        |
        |-- lead complete --> Get Leads -> Generate Lead ID -> Append Lead
        |                      -> Mark Session Completed -> Send Confirmation
        |
        '-- fields missing --> Save Session -> Which Question?
                                  |-- service  -> Ask Service Buttons
                                  |-- urgency  -> Ask Urgency Buttons
                                  '-- other    -> Send Follow-Up Question
The bot asks questions in this order and only asks for the first missing field:

Name 2. Phone number 3. Business type 4. Service 5. Budget 6. Timeline 7. Urgency
Example conversation
Bot: Welcome to Lead Qualifier! 👋 What is your name? 😊 Customer: Amaka Bot: Thanks Amaka! What is your phone number? 📱 Customer: 0801 234 5678 Bot: Thanks Amaka! What type of business do you run? 🏢 Customer: Fashion brand Bot: Thanks Amaka! What service are you looking for? 🛠️ (buttons) Customer: (taps "Ecommerce website") Bot: Thanks Amaka! What is your estimated budget for the project? 💰 Customer: ₦500k Bot: Thanks Amaka! When would you like the project completed? 📅 Customer: Within 3 weeks Bot: Thanks Amaka! How urgent is this project? ⏰ (Low / Medium / High buttons) Customer: (taps "High") Bot: Thanks! ✅ I've received your project details. Our team will review your request and get back to you soon. 🙌

Requirements
An n8n instance (self-hosted or cloud) that Telegram can reach over HTTPS
A Telegram bot token from @BotFather
A Google account with access to Google Sheets (OAuth2 credential in n8n)
A Gemini API key from Google AI Studio
Google Sheet setup
Create a spreadsheet named AI Lead Qualifier with two tabs. Header names must match exactly, because the workflow reads columns by name.

Leads (completed leads)

Lead ID	Business Type	Service Needed	Budget	Timeline	Urgency	Telegram User ID	Telegram Username	Original Message	Created At	Customer Name	Phone Number
Sessions (conversations in progress)

Telegram User ID	State	Business Type	Service Needed	Budget	Timeline	Urgency	Updated At	Original Message	Customer Name	Phone Number
Tips:

Format the Telegram User ID and Phone Number columns as Plain text (Format → Number → Plain text), so long IDs and leading zeros are kept.
Column order does not matter, but the names do. After renaming or reordering columns, click the refresh icon in each Google Sheets node.
A customer's data is always kept separate: the Telegram User ID identifies each session.
Session states
new, waiting_for_name, waiting_for_phone, waiting_for_contact, waiting_for_business_type, waiting_for_service, waiting_for_budget, waiting_for_timeline, waiting_for_urgency, completed

Setup
Create the sheet as described above.
Create credentials in n8n:
Telegram API: your bot token.
Google Sheets OAuth2.
Header Auth named Gemini API Key: Name = x-goog-api-key, Value = your Gemini API key.
Import the workflow (menu → Import from File), then add the button nodes and thank-you nodes by pasting their JSON onto the canvas (Ctrl+V).
Select your spreadsheet and tab in every Google Sheets node (Find Session, Save Session, Get Leads, Append Lead, Mark Session Completed).
Fill in the Google Sheets field mappings (the table below), then click refresh on each node.
Pick the Telegram credential in every Telegram node.
Telegram Trigger: set Trigger On to Message and Callback Query.
Check the Gemini model in the URL of the Gemini Extraction node. Model names change, so copy a current model code from Google AI Studio.
Turn on Retry On Fail (3 tries, 2000 ms) for every Telegram send node.
Save and Publish the workflow.
Google Sheets field mappings
Node	Column	Value
Save Session	Telegram User ID (match column)	{{ $json.telegram_user_id }}
State	{{ $json.state }}
Customer Name	{{ $json.customer_name ?? '' }}
Phone Number	{{ $json.phone_number ?? '' }}
Business Type	{{ $json.business_type ?? '' }}
Service Needed	{{ $json.service_needed ?? '' }}
Budget	{{ $json.budget ?? '' }}
Timeline	{{ $json.timeline ?? '' }}
Urgency	{{ $json.urgency ?? '' }}
Updated At	{{ $now.toISO() }}
Original Message	{{ $json.original_message }}
Append Lead	Lead ID	{{ $json.lead_id }}
Customer Name, Phone Number, Business Type, Service Needed, Budget, Timeline, Urgency	{{ $json.<field_name> }}
Telegram User ID, Telegram Username	{{ $json.telegram_user_id }}, {{ $json.telegram_username }}
Original Message, Created At	{{ $json.original_message }}, {{ $json.created_at }}
Mark Session Completed	Telegram User ID (match column)	{{ $('Generate Lead ID').first().json.telegram_user_id }}
State	completed
Updated At	{{ $now.toISO() }}
All other columns	leave empty
Code nodes
Node	Purpose
Extract Telegram Data	Reads the user ID, username, chat ID and text, for both messages and button taps
Prepare Session And Prompt	Loads the saved session, builds the Gemini prompt, detects thank-you messages
Parse And Validate JSON	Checks Gemini's reply, cleans values, validates phone numbers and urgency
Merge And Check Fields	Merges data, finds the next missing field, builds the reply text
Generate Lead ID	Creates L-001, L-002, ... from the existing Leads
Customising
Question order or wording: edit the order list and the questions object at the top of Merge And Check Fields.
Ask name and phone together: set ASK_TOGETHER = true in the same node.
Button choices: edit the buttons in Ask Service Buttons and Ask Urgency Buttons. The button label is sent to the AI as the customer's answer.
Extraction behaviour: edit the system prompt in Prepare Session And Prompt.
/start always begins a fresh conversation. A greeting in the middle of a chat gets "Welcome back!" and the pending question again.
Error handling
Problem	What happens
Gemini fails or returns invalid JSON	Nothing is saved, and the customer is asked to rephrase
Non-text message (photo, voice, sticker)	The customer is asked to type their details
Saving a lead fails	The customer is asked to resend, and the session stays open so the next message retries
Telegram connection drops	The send is retried up to 3 times
Testing
Clear the data rows in Sessions (keep the header row).
Message the bot Hi and answer each question.
Check the Leads tab for a new row.
Try one message with everything: "I'm Amaka, 08012345678. I own a fashion brand and need an ecommerce website. Budget ₦500k, within 3 weeks, very urgent." The lead should be saved immediately.
Test with a second Telegram account to confirm sessions never mix.
Troubleshooting
Symptom	Likely cause and fix
Bot repeats the same question	The session isn't found or saved. Check that the header is exactly Telegram User ID and that the field mappings in Save Session are filled in
"Column names were updated after the node's setup"	Click the refresh icon in that Google Sheets node
"Referenced node doesn't exist"	A node was renamed. Names must match exactly, for example Extract Telegram Data
Unexpected identifier in a Code node	Stray text was copied into the code box. Delete anything after the last }];
ERR_INVALID_HTTP_TOKEN on Gemini	The Header Auth Name is wrong. It must be exactly x-goog-api-key
Gemini 400 "unexpected model name format" or 404	Fix the model code in the Gemini Extraction URL
Test data keeps replaying	A node is pinned. Unpin it
Live bot behaves differently from the editor	You edited but did not Publish
Buttons do nothing	Add Callback Query to the Telegram Trigger, and connect Answer Button Press onward to Has Text?
ECONNRESET when sending	A network drop. Turn on Retry On Fail and check the server's internet access
Phone numbers lose the leading zero	Format the Phone Number column as Plain text
Limitations (Version 1)
Sessions never expire. An unfinished chat stays in the Sessions tab until the customer returns.
Lead IDs are made by counting existing leads, so two leads completed at the same second could clash.
Only text is supported (no voice or images).
Built for one Telegram bot with one workflow.
Intentionally not included in Version 1: Supabase/PostgreSQL, RAG or vector databases, CRM integrations, payments, voice messages, and multiple AI models.

Possible next steps
Notify the sales team (Telegram or email) when a lead is saved.
Expire old sessions automatically.
Score leads by budget and urgency.
Move storage to a database when volume grows.

