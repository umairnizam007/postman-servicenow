# postman-servicenow


### Simulating Inbound Email in ServiceNow Using Postman

- **Feasibility**: It is possible to test inbound email actions in ServiceNow without sending actual emails, especially useful in Personal Developer Instances (PDIs) where real inbound/outbound emails are disabled. This involves using Postman to make REST API calls to insert a simulated email record.
- **Approach**: Use ServiceNow's Table REST API to create a record in the `sys_email` table with necessary fields to mimic a received email. Then, optionally insert an event in the `sysevent` table to trigger processing if the inbound action doesn't fire automatically.
- **Limitations**: This method simulates email receipt but may not fully replicate real SMTP/IMAP processing, such as attachment handling or exact header parsing. It works best for testing logic in inbound actions, but test with real emails in production environments when possible.
- **Prerequisites**: Access to a ServiceNow instance with admin or appropriate roles (e.g., rest_service). Basic authentication or OAuth credentials for API calls. Knowledge of user sys_ids for senders.

#### Step-by-Step Guide
1. **Set Up Authentication in Postman**:
   - Use Basic Auth: Encode your ServiceNow username and password as Base64 (e.g., `username:password`).
   - Alternatively, set up OAuth 2.0 if your instance requires it.
   - Base URL: `https://<your-instance>.service-now.com/api/now/table/`.

2. **Create the Email Record (POST to sys_email)**:
   - Method: POST
   - URL: `https://<your-instance>.service-now.com/api/now/table/sys_email`
   - Headers:
     - `Content-Type: application/json`
     - `Accept: application/json`
     - `Authorization: Basic <base64-credentials>`
   - Body (JSON): Use fields like those below. Adjust values based on your test scenario. The `headers` field should include key email headers as a multi-line string to help parsing.
     ```json
     {
       "type": "received",
       "mailbox": "received",
       "uid": "N/A",
       "content_type": "text/html",
       "user": "<sender_sys_id>",
       "user_id": "<sender_sys_id>",
       "u_sender_address": "sender@example.com",
       "importance": "",
       "notification_type": "SMTP",
       "recipients": "recipient@yourinstance.com",
       "body_text": "This is the plain text body of the email.",
       "body": "<html><body>This is the HTML body if needed.</body></html>",
       "subject": "Test Inbound Email",
       "state": "received",
       "received_on": "2025-09-22 12:00:00",
       "headers": "Received: by example.com\nDate: Mon, 22 Sep 2025 12:00:00 GMT\nFrom: sender@example.com\nTo: recipient@yourinstance.com\nMessage-ID: <unique-id@example.com>\nMIME-Version: 1.0\nContent-Type: text/html; charset=utf-8"
     }
     ```
   - Send the request. Note the `sys_id` from the response (e.g., under `result.sys_id`).

3. **Trigger Processing (POST to sysevent, if needed)**:
   - If the inbound action doesn't trigger after inserting the email, create an event to simulate the read process.
   - Method: POST
   - URL: `https://<your-instance>.service-now.com/api/now/table/sysevent`
   - Headers: Same as above.
   - Body (JSON):
     ```json
     {
       "name": "email.read",
       "parm1": "<email_sys_id_from_previous_step>"
     }
     ```
   - This queues the email for processing, mimicking the system's handling of inbound emails.

4. **Verify Results**:
   - Navigate to `System Mailboxes > Received` in ServiceNow to check if the email appears.
   - Check `System Logs > Emails` for logs.
   - If an inbound action is configured (e.g., to create an incident), verify the target record (e.g., in `incident` table).
   - If issues arise, ensure the `headers` and `content_type` match, and the sender's email resolves to a valid `sys_user` record.

#### Common Issues and Tips
- **Inbound Action Not Triggering**: Double-check `type` is "received" and add the `sysevent` step. Some setups require a valid `From` address matching a user.
- **PDI Restrictions**: Emails are disabled in PDIs, so this API method is essential. For non-PDIs, consider real email testing via Gmail or Zoho integration for fuller validation.
- **Headers Customization**: Inspect a real email's source (e.g., in Gmail, "Show Original") to copy realistic headers. Minimal headers may work, but include From, To, Date, and Content-Type for better simulation.
- **Security**: Ensure API access is scoped; use roles like `rest_api_explorer` for testing.
- **Alternatives**: For simple tests, manually insert records via the ServiceNow UI, but Postman allows automation and scripting.

---

Inbound email actions in ServiceNow automate responses to incoming emails, such as creating or updating records like incidents or requests. While actual inbound emails are handled via configured email servers (SMTP/POP3/IMAP), testing often requires simulation to avoid dependency on external mail services, especially in restricted environments like Personal Developer Instances (PDIs).

#### Overview of Inbound Email Processing
ServiceNow processes inbound emails by polling configured mailboxes, inserting received messages into the `sys_email` table, and triggering associated inbound actions or flows. Key tables involved:
- **sys_email**: Stores email details (e.g., subject, body, headers).
- **sysevent**: Handles events like `email.read` to initiate processing.
- **sys_email_log**: Logs actions taken on emails.

Inbound actions are scripts or flows that parse the email and perform operations, such as:
- Creating a new record (e.g., incident from subject/body).
- Updating existing records (e.g., adding comments to a replied incident).
- Replying automatically.

For example, a basic inbound action might check the subject for keywords and insert a problem record if conditions match.

#### Simulating with REST API (Detailed)
Since Postman excels at HTTP requests, leverage ServiceNow's Table REST API for simulation. This bypasses email servers by directly inserting data.

**Endpoint Details**:
- Base: `https://<instance>.service-now.com/api/now/table/<table_name>`
- Authentication: Basic (username:password Base64-encoded) or OAuth 2.0. Response codes: 201 (Created), 400 (Bad Request), 401 (Unauthorized).
- Request Format: JSON body with field-value pairs. Fields are case-sensitive and must match the table schema.

**sys_email Table Key Fields** (Based on Common Usage):
| Field Name | Type | Description | Example Value | Required for Inbound? |
|------------|------|-------------|---------------|-----------------------|
| type | Choice | Email type (e.g., received, sent) | "received" | Yes |
| mailbox | String | Mailbox category | "received" | Yes |
| uid | String | Unique ID (optional) | "N/A" | No |
| content_type | String | MIME type | "text/html" or "multipart/mixed" | Yes, for body parsing |
| user | Reference (sys_user) | Sender reference | Sys_id of user | Yes, for from-address resolution |
| user_id | String | Sender email or ID | "sender@example.com" | Yes |
| u_sender_address | String | Sender email | "sender@example.com" | Yes |
| recipients | String | To/CC addresses | "instance@email.com" | Yes |
| body_text | String | Plain text body | "Test body text" | Optional (if using body) |
| body | String | HTML body | "<p>Test HTML</p>" | Optional (if using body_text) |
| subject | String | Email subject | "Test Subject" | Yes |
| state | Choice | Processing state | "received" | Yes |
| received_on | Date/Time | Receipt timestamp | "2025-09-22 12:00:00" | Recommended |
| headers | String | Email headers (multi-line) | "From: sender@example.com\nTo: recipient@example.com\n..." | Yes, for accurate parsing |
| importance | String | Priority | "normal" | No |
| notification_type | String | Protocol | "SMTP" | No |

**Example Postman Collection Setup**:
- Create a collection in Postman.
- Add a POST request for `sys_email` with the JSON body above.
- Use variables for instance URL, credentials, and sys_ids.
- Chain requests: Use Postman's scripting to capture the email sys_id and use it in the sysevent POST.

**sysevent Table for Triggering**:
| Field Name | Type | Description | Example Value |
|------------|------|-------------|---------------|
| name | String | Event name | "email.read" |
| parm1 | String | Parameter (email sys_id) | "<sys_id>" |
| process_on | Date/Time | When to process | Current time (auto-set) |

This event queues the email for the system's email processor.

#### Testing in PDIs vs. Production
In PDIs, real emails are blocked since mid-2023, making API simulation the primary method. Users report success with the insert-and-trigger approach, though occasional errors like "fromAddress is not resolved" may occur if the sender email doesn't match a sys_user record. In production, combine with real tests using external mail servers (e.g., Gmail integration via app passwords).

#### Advanced Considerations
- **Attachments**: Simulate by inserting into `sys_attachment` and linking via `sys_email_attachment`.
- **Flows vs. Actions**: For modern instances, use Flow Designer with inbound email triggers, which reference `sys_email` fields similarly.
- **Error Handling**: Check system logs for issues like mismatched content_type. If actions fail, use the "Reprocess Email" UI action on the sys_email record.
- **Best Practices**: Start with minimal fields, add complexity (e.g., watermarks for replies). Prioritize official docs for release-specific changes (e.g., Zurich vs. Washington DC).
- **Alternatives to Postman**: Use cURL, REST API Explorer in ServiceNow, or scripted inserts via background scripts.

This method has been validated in community discussions for troubleshooting and development, ensuring reliable testing without external dependencies.

#### Key Citations
- [Manually Insert New Email Record for Inbound Email testing](https://www.servicenow.com/community/developer-forum/manually-insert-new-email-record-for-inbound-email-testing-is-it/m-p/2508506)
- [Is there a way of testing Inbound Email Actions on PDIs?](https://www.reddit.com/r/servicenow/comments/14xsxdi/is_there_a_way_of_testing_inbound_email_actions/)
- [Table API](https://www.servicenow.com/docs/bundle/zurich-api-reference/page/integrate/inbound-rest/concept/c_TableAPI.html)
- [System email log and mailboxes](https://www.servicenow.com/docs/bundle/zurich-platform-administration/page/administer/time/reference/r_EmailLogs.html)
- [Accessing email object variables](https://www.servicenow.com/docs/bundle/zurich-platform-administration/page/administer/notification/reference/r_AccessingEmailObjsWithVars.html)
- [Create an inbound email action](https://www.servicenow.com/docs/bundle/zurich-platform-administration/page/administer/notification/task/t_CreatingAnInboundEmailAction.html)
