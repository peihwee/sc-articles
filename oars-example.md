# OARS Report Example

In OWASP's WebGoat, the "Bypass Front-End Restrictions" exercise (found under the Client Side module) challenges you to submit data that violates the visual constraints or validation rules enforced on the web page.

The lesson proves a fundamental rule of secure coding: Client-side validation is for user experience, not for security.

Here is a walkthrough of the exercise and how it maps to an OARS incident and remediation report.

------------------------------
## The WebGoat Exercise Scenario
The target page displays a form with several field restrictions that block you from typing unexpected inputs: 

   1. A Dropdown Select Menu: Only has Option 1 and Option 2.
   2. Radio Buttons: Forces an option choice.
   3. A Checkbox: Toggles on/off.
   4. A Short Text Field: Enforces a maximum character length (e.g., maxlength="5").

The Goal: Force the application to accept illegal inputs (e.g., submitting an option that doesn't exist or a text string longer than 5 characters).

## How to Solve it (Two Methods):

* Method A (Browser DevTools): Right-click an element (like the text box), choose Inspect, and manually modify the DOM. For instance, change maxlength="5" to maxlength="20", or alter an option value attribute to `<option value="illegal_option">`. Hit submit.
* Method B (Interception Proxy): Type valid data into the fields, turn on an interception proxy like Burp Suite or OWASP ZAP, click submit, and alter the raw HTTP POST request parameters in transit before forwarding it to the server. 

------------------------------
## OARS Report for the WebGoat Bypass

| Phase | Description | WebGoat Exercise Application |
|---|---|---|
| O: Observe | What happened normally? | An attacker evaluated a profile update form. Normally, the UI enforces restriction rules: the text field only permits up to 5 characters, and the dropdown only contains preset values option1 and option2. |
| A: Act | What authorized action did you take? | The testing team intercepted the submission request via Burp Suite[](https://portswigger.net/burp) and utilized browser Developer Tools to temporarily unlock the frontend DOM constraints. |
| R: Result | What changed? | The application backend processed the modified request without an error, successfully saving an illegal 15-character string and an arbitrary value (option3) directly into the backend state. This proved the backend server had zero validation checks of its own. |
| S: Secure | How should the application defend? | Implement mirror validation routines directly on the server controller handling the data. The backend must explicitly verify input lengths and reject any option values that do not match an established server-side allowlist. |

------------------------------
## Secure Coding: The "S" Phase Code Implementation
To fulfill the Secure tier of the framework, the application must handle the data safely once it reaches the server.
Vulnerable Backend Code (What WebGoat does):

```
// The server blindly trusts incoming client data
app.post('/api/update-profile', (req, res) => {
    const { shortInput, selectOption } = req.body;
    
    // CRITICAL FLAW: Directly updates database without validation
    db.saveProfile(shortInput, selectOption); 
    res.status(200).send("Profile updated!");
});
```

Secure Backend Code (The Fix):

```
// The server treats all client data as untrusted
app.post('/api/update-profile', (req, res) => {
    const { shortInput, selectOption } = req.body;
    const allowedOptions = ['option1', 'option2'];

    // 1. Enforce length restrictions on the server
    if (!shortInput || shortInput.length > 5) {
        return res.status(400).send("Error: Input exceeds maximum allowed length.");
    }

    // 2. Enforce strict allowlists for structural options
    if (!allowedOptions.includes(selectOption)) {
        return res.status(400).send("Error: Invalid option selection detected.");
    }

    // Process safely only after validations pass
    db.saveProfile(shortInput, selectOption);
    res.status(200).send("Profile securely updated!");
});
```