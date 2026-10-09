# Parloa Substitute Receipt Creator

An internal tool for Parloa employees. When an original receipt is missing, it creates a self-issued **Substitute Receipt** as a PDF. Fill in the form, sign, preview the document, then download it or send it by email.

It is a single file, `index.html`, and runs entirely in the browser. There is no build step and no backend. Form data and signatures stay on your device. The only exception is when you choose to send an email: the PDF then goes through EmailJS.

## Opening the app

**Option A: double-click.** Open `index.html` in Chrome, Edge, Firefox or Safari. You need an internet connection because the libraries and fonts load from a CDN.

**Option B: host it.** Upload the folder to any static host, such as an internal web server, GitHub Pages, Netlify or an S3 bucket. To try it locally:

```bash
npx serve .            # or: python3 -m http.server 8000
```

Then open the printed URL.

The Parloa logo is embedded in `index.html`, so the file works on its own. You can share just that one file.

### What's inside

| Library | Version | Purpose |
| --- | --- | --- |
| [jsPDF](https://github.com/parallax/jsPDF) | 4.2.1 | Builds the A4 PDF |
| [html2canvas](https://html2canvas.hertzen.com/) | 1.4.1 | Renders the document at 3× resolution for sharp text |
| [signature_pad](https://github.com/szimek/signature_pad) | 5.1.4 | Signature drawing canvas |
| [@emailjs/browser](https://www.emailjs.com/docs/sdk/installation/) | 4.4.1 | Sends the email with the PDF attached |

All four load from jsDelivr with pinned versions. Fonts (Inter and Newsreader) load from Google Fonts.

### Features

- Receipt numbers are suggested in the format `SR-2026-0001`. The last number you used is stored in `localStorage` and moves up after each successful download or send. You can always edit the number.
- Drafts are saved automatically while you type. Use **Clear form** to start over; it offers **Undo**. Once a receipt has been downloaded or sent, the next visit restores only your name, department and email, together with a new number. The signature and the confirmation checkbox are never saved.
- Inline validation: when you press **Generate preview**, the page scrolls to the first problem.
- For amounts above 150 you see a soft warning, but you can still continue.
- Signature: draw it (high-DPI, with Undo and Clear) or upload a PNG/JPG. A white background on an uploaded image is removed automatically.
- File names look like `Substitute-Receipt_SR-2026-0001_2026-10-09.pdf`.

## Setting up email (EmailJS)

The app sends email through [EmailJS](https://www.emailjs.com/), so no server is needed. Until it is configured, the **Send** button shows a clear message, and **Download PDF** keeps working.

1. **Create an account** at <https://www.emailjs.com/> and sign in.
2. **Add an email service.** Go to *Email Services* → *Add New Service*, choose your provider (for example Gmail, or Outlook/Microsoft 365 for a shared Accounting mailbox), connect it, and click *Create Service*. Copy the **Service ID**, for example `service_ab12cd3`.
3. **Create a template.** Go to *Email Templates* → *Create New Template*.
   - **Settings / To Email:** `{{to_email}}`
   - **Reply To:** `{{reply_to}}`
   - **Subject:** `Substitute Receipt {{receipt_number}} – {{employee_name}} – {{amount}}`
   - **Content**, for example:
     ```
     Hello,

     please find attached the Substitute Receipt {{receipt_number}}.

     Employee:      {{employee_name}} ({{department}})
     Date:          {{expense_date}}
     Payee:         {{payee}}
     Amount:        {{amount}}
     Reason:        {{reason}}
     Description:   {{description}}
     ```
   - **Attachments tab** → *Add Attachment* → **Variable Attachment**:
     - *Parameter name:* `pdf_attachment` (must match `CONFIG.emailjs.attachmentParam`)
     - *Filename:* `{{file_name}}`
     - *Content type:* PDF

     The app sends the PDF in this parameter as a base64 data URL (`data:application/pdf;base64,...`).
   - Save, then copy the **Template ID**, for example `template_x9y8z7`.
4. **Copy your public key** from *Account* → *General* → *Public Key*.
5. **Paste the keys into `CONFIG`** at the top of the `<script>` in `index.html`:
   ```js
   emailjs: {
     publicKey: 'AbCdEfGhIjKlMnOp',
     serviceId: 'service_ab12cd3',
     templateId: 'template_x9y8z7',
     attachmentParam: 'pdf_attachment',
     defaultRecipient: 'accounting@parloa.com', // optional: used when the recipient field is empty
   },
   ```
6. **Restrict usage (recommended).** In EmailJS, go to *Account* → *Security* and add the domain where you host the app to the allow-list. The public key appears in the page source, so this stops it from being used elsewhere.

**Template parameters sent:** `to_email`, `reply_to`, `receipt_number`, `employee_name`, `employee_email`, `department`, `amount` (already formatted, e.g. `1.234,56 €`), `expense_date`, `payee`, `description`, `reason`, `file_name`, plus the attachment parameter.

**Attachment size:** the PDF sent by email is rendered at a slightly lower resolution and is usually around 150–300 KB. EmailJS limits the request size depending on your plan, and the free plan's limit is far below that. Check the current limits on the EmailJS pricing page; you will probably need a paid plan to send attachments. If a send fails, the app shows the error and you can still download the PDF.

## Customising

All settings are in the `CONFIG` object at the top of the script in `index.html`.

### Company name and address

```js
company: {
  name: 'Parloa GmbH',
  address: ['Schönhauser Allee 9', '10119 Berlin', 'Germany'], // one entry per line
},
```

Other settings: `numberPrefix` (default `SR`), `defaultCurrency` (default `EUR`), `currencies` (the options in the dropdown) and `highAmountThreshold` (default `150`).

### Swapping the logo

The dark logo is embedded as a PNG data URL (`PARLOA_LOGO_PNG`, defined just above `CONFIG`) and assigned to `CONFIG.logo.dataUrl`. The PDF uses it as is, and the dark app header shows it recoloured to white. Because it is embedded, it also renders in the PDF when the file is opened by double-click. Browsers block pages opened via `file://` from reading separate image files into the PDF renderer, which is why a separate file isn't used.

To use a different logo:

1. Create a dark logo on a transparent background (PNG or SVG). `parloa-logo.png` and `parloa-logo-white.png` in this repo are the current versions.
2. Convert it to a data URL and replace the value of `PARLOA_LOGO_PNG`:
   ```bash
   echo "data:image/png;base64,$(base64 -w0 parloa-logo.png)"   # macOS: base64 -i parloa-logo.png
   ```
   Alternatively, set `PARLOA_LOGO_PNG` to `''` and point `CONFIG.logo.src` to a file next to `index.html`. That only shows up in the PDF when the app is hosted, not when it is opened by double-click.
3. Optional: set `CONFIG.logo.srcOnDark` to a dedicated white logo for the app header.

If no logo can be loaded, the app and the PDF show the text wordmark "Parloa".

## Notes

- The PDF is rendered as a high-resolution image, so text in it cannot be selected. The PDF metadata (title, author) is set.
- Normal content fits on one A4 page. If every text field is filled to its limit, the page is scaled down slightly so it still fits on one page.
- The PDF footer states that the document is a self-issued substitute receipt in accordance with GoBD and that no input VAT deduction (Vorsteuerabzug) can be claimed on its basis.
