# RoamCX contact form backend

`contact-form.yaml` deploys everything behind the "Get in touch" form:

```
browser  ──POST /contact──▶  HTTP API Gateway  ──▶  Lambda  ──┬──▶  DynamoDB  (every submission)
                                                              └──▶  SES       (notification email)
```

## Before you deploy

Pick **one region** and use it for both SES and this stack — SES identities are
regional, and the Lambda calls SES in its own region. `eu-west-2` (London) is
the natural choice here.

In SES, verify:

- `priyatam.piyush@roamcx.com` — the sender, and a recipient
- `jas.dhesi@roamcx.com` — the other recipient

While the SES account is still in the **sandbox**, every recipient must be a
verified identity too. Request production access when you're ready to remove
that restriction (it doesn't affect this stack).

## Deploy

```bash
aws cloudformation deploy \
  --region eu-west-2 \
  --stack-name roamcx-contact-form \
  --template-file infra/contact-form.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
      SenderEmail=priyatam.piyush@roamcx.com \
      RecipientEmails='jas.dhesi@roamcx.com\,priyatam.piyush@roamcx.com' \
      AllowedOrigin=https://roamcx.com
```

`CAPABILITY_NAMED_IAM` is required because the Lambda execution role has a
fixed name. Note the escaped comma in `RecipientEmails` — the CLI splits
parameter overrides on unescaped commas.

On an **update**, `aws cloudformation deploy` keeps the previous value of any
parameter you don't pass — changing a `Default` in the template has no effect
on an existing stack. Always pass the parameter you want to change explicitly.

Then grab the endpoint:

```bash
aws cloudformation describe-stacks \
  --region eu-west-2 \
  --stack-name roamcx-contact-form \
  --query 'Stacks[0].Outputs' --output table
```

## Wire up the site

Paste the `ContactEndpoint` output into `index.html`:

```js
const CONTACT_ENDPOINT = 'https://xxxxxxxx.execute-api.eu-west-2.amazonaws.com/prod/contact';
```

`AllowedOrigin` must exactly match the origin the site is served from —
scheme + host, no trailing slash, and `www.` counts as a different origin.
A mismatch shows up as a CORS error in the browser console, not a server error.

## Parameters

| Parameter | Default | Notes |
|---|---|---|
| `SenderEmail` | `priyatam.piyush@roamcx.com` | Must be a verified SES identity |
| `RecipientEmails` | both addresses | Comma-separated |
| `AllowedOrigin` | `https://roamcx.com` | Use `http://localhost:8000` for local testing |
| `SubmissionRetentionDays` | `0` | `0` = keep forever; otherwise TTL-deletes after N days |
| `LogRetentionDays` | `30` | CloudWatch Logs retention |
| `ThrottleRateLimit` / `ThrottleBurstLimit` | `5` / `10` | Per-stage request throttling |

## Reading submissions

Every submission is stored whether or not the email sends. Newest first, no scan:

```bash
aws dynamodb query \
  --region eu-west-2 \
  --table-name roamcx-contact-submissions \
  --index-name byDateIndex \
  --key-condition-expression 'recordType = :t' \
  --expression-attribute-values '{":t":{"S":"submission"}}' \
  --no-scan-index-forward --max-items 20
```

The table has point-in-time recovery on and `DeletionPolicy: Retain`, so
deleting the stack leaves your submissions intact.

## If mail doesn't arrive

The site says "thanks" whenever the submission was **stored**, even if SES
failed — so check the logs rather than the form:

```bash
aws logs tail /aws/lambda/roamcx-contact-handler --region eu-west-2 --follow
```

`ERROR send_email` there almost always means an unverified identity (sender,
or a recipient while in the sandbox). The stored item is still in DynamoDB.

## Behaviour worth knowing

- **Reply-To** is set to the enquirer's address, so hitting reply in your inbox
  goes straight to them.
- A hidden `website` honeypot field silently drops bot submissions — they get a
  `200` but nothing is stored or emailed.
- Server-side validation mirrors the form's `required` attributes and caps
  field lengths; the message body is HTML-escaped before it goes into the email.
