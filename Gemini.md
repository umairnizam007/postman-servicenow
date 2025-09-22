## Simulating Inbound Emails in ServiceNow with Postman: A Proof of Concept

**New York, NY – September 22, 2025** – For developers and administrators working with ServiceNow, testing and debugging inbound email functionalities can be a cumbersome process. This proof of concept (POC) outlines a streamlined method for simulating inbound emails using Postman, enabling rapid testing of inbound email actions and flows without the need for an external email client.

This guide provides a step-by-step approach to configuring both your ServiceNow instance and Postman to create and process simulated inbound emails, thereby accelerating development and ensuring the reliability of your email-based workflows.

### **1. ServiceNow Configuration: Preparing for Inbound Emails**

The first phase involves setting up a dedicated user and an inbound email action within your ServiceNow instance.

**1.1. Create a Dedicated Integration User:**

To ensure secure and auditable integrations, it is best practice to create a specific user account for this purpose.

*   Navigate to **User Administration > Users** and create a new user.
*   Assign this user the `rest_service` and `itil` roles. The `rest_service` role grants the ability to interact with the REST API, while `itil` is often necessary to create records like incidents.
*   Ensure the user has a password and is marked as "Active."

**1.2. Define an Inbound Email Action:**

Inbound email actions are the core logic that ServiceNow executes upon receiving an email.

*   Navigate to **System Policy > Email > Inbound Actions**.
*   Create a new inbound action.
*   **Target Table:** Specify the table you want the inbound email to affect, for example, the "Incident" table.
*   **Action Type:** For this POC, select "Record Action."
*   **Conditions:** Define the trigger conditions. For instance, you can configure the action to run if the email subject contains the word "Incident."
*   **Actions (Script):** In the "Actions" tab, provide a script to process the email and create a record. Below is a sample script to create an incident from an email:

```javascript
(function runAction(current, event, email, logger, classifier) {

  // Create an incident record
  var incident = new GlideRecord('incident');
  incident.initialize();
  incident.short_description = email.subject;
  incident.description = email.body_text;
  incident.caller_id = gs.getUserID(); // Sets the caller to the integration user
  incident.insert();

})(current, event, email, logger, classifier);
```

### **2. Postman Configuration: Simulating the Inbound Email**

With ServiceNow configured, the next step is to set up Postman to send a request that simulates an inbound email. This is achieved by making a POST request to the `sys_email` table in ServiceNow.

**2.1. Request Setup:**

*   **HTTP Method:** `POST`
*   **URL:** `https://<your-instance>.service-now.com/api/now/table/sys_email`
    *   Replace `<your-instance>` with your ServiceNow instance name.

**2.2. Authentication:**

*   In the "Authorization" tab, select "Basic Auth."
*   Enter the username and password of the dedicated integration user you created in step 1.1.

**2.3. Headers:**

*   `Content-Type`: `application/json`
*   `Accept`: `application/json`

**2.4. Request Body:**

The body of the request must contain specific fields to accurately mimic an inbound email.

*   In the "Body" tab, select "raw" and "JSON (application/json)."
*   Construct a JSON payload with the following fields:

```json
{
  "subject": "New Incident: Critical network failure",
  "body": "This is a test email to create an incident.",
  "recipients": "your-instance@service-now.com",
  "type": "received",
  "headers": "Content-Type: text/plain; charset=UTF-8\nMIME-Version: 1.0",
  "content_type": "text/plain",
  "state": "received"
}
```

**Key fields in the request body:**

*   **`subject` and `body`:** The content of your simulated email.
*   **`recipients`:** The email address of your ServiceNow instance.
*   **`type`:** Must be set to `"received"` to indicate it's an inbound email.
*   **`headers`:** Include basic email headers.
*   **`content_type`:** Specifies the format of the email body.
*   **`state`:** Set to `"received"` to place the email in the processing queue.

### **3. Execution and Verification**

**3.1. Send the Request:**

*   Click the "Send" button in Postman. A successful request will return a `201 Created` status code with the details of the newly created `sys_email` record.

**3.2. Verify in ServiceNow:**

*   **Check the Email Log:** Navigate to **System Logs > Emails**. You should see the email you sent from Postman in the log with the type "received."
*   **Confirm Record Creation:** Navigate to the target table of your inbound action (e.g., **Incident > All**). A new incident should be created with the subject and body from your Postman request.

By following this proof of concept, you can efficiently test and refine your inbound email workflows in ServiceNow, leading to more robust and reliable automations. This method provides a powerful alternative to traditional email testing, offering greater control and speed for developers.
