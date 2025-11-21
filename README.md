# VAPI Outbound Caller Workflow

An n8n workflow that initiates intelligent AI-powered outbound calls through VAPI, dynamically fetching available appointment slots from GoHighLevel and passing them to the AI assistant for natural conversation-based booking.

## Overview

This workflow enables automated outbound calling for appointment booking by:
- Receiving lead information via webhook
- Fetching real-time available appointment slots from GoHighLevel
- Formatting slot data in a human-readable format
- Initiating VAPI AI voice calls with personalized context
- Enabling the AI to book appointments during the call

## Workflow Architecture

```
Webhook Trigger → Extract Lead Data → Wait (2s) → Fetch Calendar Slots
                                                         ↓
                                                   Format for AI
                                                         ↓
                                                   Wait (10s)
                                                         ↓
                                                   Initiate VAPI Call
```

## Prerequisites

- n8n instance (self-hosted or cloud)
- VAPI account with phone number and assistant configured
- GoHighLevel account with API access
- GoHighLevel Calendar ID

## Key Features

### 🎯 Intelligent Slot Formatting
- Fetches next 10 days of availability
- Converts UTC timestamps to Pacific Time (America/Los_Angeles)
- Groups slots by date with formatted times (12-hour format)
- Provides first 3 days of availability to AI

### 🤖 Context-Aware AI Calls
- Passes customer name, phone, and email to VAPI
- Injects real-time availability into conversation
- Enables natural appointment booking dialogue

### ⏱️ Strategic Timing
- 2-second delay before fetching slots (system preparation)
- 10-second delay before call initiation (optimal timing)

## Configuration

### 1. GoHighLevel Setup

**Calendar ID:** `x4HMpTaDe7tp8WrQWRr6`  
**Time Zone:** America/Los_Angeles (Pacific Time)

Update the calendar ID in the "Get Available Slots" node:
```
/calendars/YOUR_CALENDAR_ID/free-slots
```

### 2. VAPI Configuration

**Phone Number ID:** `a4f3156c-b8d8-4f3e-906f-6bbf43586189`  
**Assistant ID:** `d069c698-c184-4ed4-beb6-31d7cd067121`

Update these in the "Make VAPI Call" node with your own IDs:
- Phone Number ID: Found in VAPI dashboard → Phone Numbers
- Assistant ID: Found in VAPI dashboard → Assistants

### 3. n8n Credentials

#### GHL API Credential
Create "GHL E&E Landscaping API" HTTP Header Auth:
- **Header Name:** `Authorization`
- **Header Value:** `Bearer YOUR_GHL_API_KEY`

#### VAPI API Credential
Create "VAPI Header Auth" HTTP Header Auth:
- **Header Name:** `Authorization`
- **Header Value:** `Bearer YOUR_VAPI_API_KEY`

### 4. Webhook Configuration

The workflow uses path: `outbound-ai-call`

Your webhook URL will be:
```
https://your-n8n-instance.com/webhook/outbound-ai-call
```

## Input Format

Send a POST request to the webhook with the following JSON structure:

```json
{
  "first_name": "John",
  "last_name": "Doe",
  "phone": "+15551234567",
  "email": "john.doe@example.com"
}
```

### Required Fields
- `first_name` (string) - Customer's first name
- `phone` (string) - Customer's phone number in E.164 format

### Optional Fields
- `last_name` (string) - Customer's last name
- `email` (string) - Customer's email address

## Workflow Nodes

### 1. Webhook
Receives lead information via POST request from external systems (CRM, forms, etc.).

### 2. Extract Lead Data
Parses incoming webhook data and normalizes fields:
- First name
- Last name (defaults to empty string if not provided)
- Phone number
- Email (defaults to empty string if not provided)

### 3. Wait Before Slots
Introduces a 2-second delay to ensure system readiness before API calls.

### 4. Get Available Slots
Fetches available appointment slots from GoHighLevel:
- **Endpoint:** `/calendars/{calendarId}/free-slots`
- **Time Range:** Current time to +10 days
- **Time Zone:** Pacific Time (America/Los_Angeles)
- **API Version:** 2021-04-15

### 5. Format Slots for AI
JavaScript code node that:
- Parses raw slot data from GoHighLevel
- Converts UTC times to Pacific Time
- Formats dates (e.g., "Monday, March 15, 2024")
- Formats times in 12-hour format (e.g., "2:30 PM")
- Groups slots by date
- Generates natural language text for first 3 days
- Combines with customer data for VAPI

**Output Example:**
```
"On Monday, March 15, 2024, we have: 9:00 AM, 10:00 AM, 2:00 PM, 3:30 PM. 
On Tuesday, March 16, 2024, we have: 10:00 AM, 11:00 AM, 1:00 PM, 4:00 PM. 
On Wednesday, March 17, 2024, we have: 9:30 AM, 11:30 AM, 2:30 PM."
```

### 6. Wait Before Call
Introduces a 10-second delay before initiating the call for optimal timing.

### 7. Make VAPI Call
Initiates the outbound AI call via VAPI API:
- Uses specified phone number ID for outbound calling
- Assigns assistant ID for the conversation
- Passes variable values to assistant context

**Variables Passed to AI:**
- `customer_name` - First name for personalization
- `customer_phone` - Phone number for record keeping
- `available_slots` - Formatted availability text

## VAPI Assistant Configuration

Your VAPI assistant should be configured with the following variables:

```json
{
  "variableValues": {
    "customer_name": "{{customer_name}}",
    "customer_phone": "{{customer_phone}}",
    "available_slots": "{{available_slots}}"
  }
}
```

### Recommended Assistant Prompt

```
You are a friendly appointment scheduler for E&E Landscaping. 

The customer's name is {{customer_name}} and their phone number is {{customer_phone}}.

Here are our available appointment times:
{{available_slots}}

Your goal is to:
1. Greet the customer warmly by name
2. Explain you're calling to schedule their landscaping consultation
3. Present the available times in a natural, conversational way
4. Help them select a convenient time
5. Confirm the appointment details

When the customer selects a time, use the bookAppointment function to finalize the booking.

Be natural, friendly, and patient. If they need to check their schedule, offer to wait or call back later.
```

### Required Function

Your VAPI assistant needs the `bookAppointment` function that connects to the [Appointment Booking Workflow](../appointment-booking):

```json
{
  "name": "bookAppointment",
  "description": "Books an appointment for the customer",
  "parameters": {
    "type": "object",
    "properties": {
      "customerName": {
        "type": "string",
        "description": "Full name of the customer"
      },
      "customerPhone": {
        "type": "string",
        "description": "Phone number of the customer"
      },
      "appointmentDateTime": {
        "type": "string",
        "description": "ISO 8601 datetime for the appointment"
      }
    },
    "required": ["customerName", "customerPhone", "appointmentDateTime"]
  },
  "serverUrl": "https://your-n8n-instance.com/webhook/9f1eb040-8388-463b-af4f-5ca49c31fa2e"
}
```

## Integration Points

### Trigger Sources
This workflow can be triggered from:
- **GoHighLevel:** Webhook automation on new contact
- **CRM Systems:** API webhooks for new leads
- **Lead Forms:** Form submissions via Zapier/Make
- **Manual Trigger:** API call from internal systems

### Connected Workflows
- **[Appointment Booking Workflow](../appointment-booking)** - Receives booking requests from VAPI during calls

## Installation

1. Import the workflow JSON into your n8n instance
2. Update GoHighLevel Calendar ID
3. Update VAPI Phone Number ID and Assistant ID
4. Configure GHL API credentials
5. Configure VAPI API credentials
6. Activate the workflow
7. Test with a sample webhook payload

## Testing

### Test via n8n UI
1. Click "Execute Workflow"
2. Send test webhook data
3. Monitor execution through each node
4. Verify VAPI call is initiated

### Test Payload
```json
{
  "first_name": "Jane",
  "last_name": "Smith",
  "phone": "+15559876543",
  "email": "jane.smith@example.com"
}
```

### End-to-End Test
1. Trigger workflow with real lead data
2. Verify AI call is initiated to the phone number
3. Have a conversation with the AI
4. Book an appointment during the call
5. Verify appointment appears in GoHighLevel calendar

## API Endpoints Used

### GoHighLevel API
- `GET /calendars/{calendarId}/free-slots` (Version 2021-04-15)
  - Fetches available appointment slots
  - Parameters: startDate, endDate (Unix timestamps in milliseconds)

### VAPI API
- `POST /call/phone`
  - Initiates outbound phone call
  - Requires: phoneNumberId, customer number, assistantId, assistantOverrides

## Time Zone Handling

This workflow uses **America/Los_Angeles (Pacific Time)** for all time operations:

```javascript
// Start date: Current time in Pacific
$now.setZone('America/Los_Angeles').toMillis()

// End date: 10 days from now in Pacific
$now.plus(10, 'days').setZone('America/Los_Angeles').toMillis()
```

To change time zones, update all instances of `'America/Los_Angeles'` in:
- Get Available Slots node (URL parameters)
- Format Slots for AI node (JavaScript code)

Common time zones:
- `'America/New_York'` - Eastern Time
- `'America/Chicago'` - Central Time
- `'America/Denver'` - Mountain Time
- `'America/Los_Angeles'` - Pacific Time

## Customization

### Adjust Availability Window
Change the 10-day window in "Get Available Slots":
```javascript
// 7 days instead of 10
$now.plus(7, 'days').setZone('America/Los_Angeles').toMillis()

// 14 days
$now.plus(14, 'days').setZone('America/Los_Angeles').toMillis()
```

### Show More/Fewer Days to AI
Modify the counter in "Format Slots for AI":
```javascript
// Show first 5 days instead of 3
if (count < 5) {
```

### Adjust Wait Times
- **Wait Before Slots:** Default 2 seconds (can reduce to 0-1 for faster execution)
- **Wait Before Call:** Default 10 seconds (recommended for human-like timing)

## Troubleshooting

### Call Not Initiated
- Verify VAPI phone number ID is correct
- Check VAPI API key is valid
- Ensure phone number is in E.164 format (+1XXXXXXXXXX)
- Verify VAPI has sufficient credits

### No Available Slots
- Check calendar ID is correct
- Verify calendar has availability configured
- Ensure time zone matches your calendar settings
- Check date range (10 days from now)

### AI Doesn't Have Slot Information
- Review "Format Slots for AI" execution output
- Verify slots are being fetched successfully
- Check that variable values are passed correctly to VAPI

### Wrong Time Zone
- All times displayed are in Pacific Time by default
- Update time zone strings in workflow if serving different region
- Verify calendar time zone matches workflow time zone

## Best Practices

### Phone Number Format
Always use E.164 format for phone numbers:
- ✅ Correct: `+15551234567`
- ❌ Incorrect: `555-123-4567`, `(555) 123-4567`, `5551234567`

### Call Timing
The 10-second wait before initiating calls:
- Allows real-time data to be current
- Creates human-like pacing
- Prevents rapid-fire calls from appearing automated

### Lead Quality
For best results, only trigger calls for:
- Qualified leads who expect contact
- Leads with valid phone numbers
- Leads in appropriate time zones (respect calling hours)

## Cost Considerations

### VAPI Costs
- Per-minute calling charges
- Failed calls still consume credits
- Consider implementing retry logic with delays

### API Rate Limits
- GoHighLevel: Standard rate limits apply
- VAPI: Check your plan's concurrent call limit

## Security Notes

- Store all API keys in n8n credentials (never in code)
- Use HTTPS for webhook endpoints
- Validate incoming webhook data
- Consider webhook authentication tokens
- Implement rate limiting if webhook is publicly accessible

## Related Workflows

- **[Appointment Booking Workflow](../appointment-booking)** - Handles appointments booked during VAPI calls
- (Add links to other related workflows)

## Workflow Flow Summary

```
1. Receive lead data via webhook
2. Extract and normalize customer information
3. Wait 2 seconds for system readiness
4. Fetch next 10 days of calendar availability
5. Format slots into natural language
6. Wait 10 seconds for optimal timing
7. Initiate VAPI AI call with customer context
8. AI conducts conversation and books appointment
9. Booking sent to Appointment Booking Workflow
```

## License

MIT License - Feel free to modify and use for your projects.

## Support

For issues or questions:
1. Check execution logs in n8n for detailed error messages
2. Verify all credentials are correctly configured
3. Test each node individually to isolate issues
4. Review VAPI call logs in VAPI dashboard

## Version History

- **v1.0** - Initial release with 10-day availability and Pacific Time support

---

**Note:** This workflow is designed to work seamlessly with the Appointment Booking Workflow. Ensure both workflows are active and properly configured for full functionality.
