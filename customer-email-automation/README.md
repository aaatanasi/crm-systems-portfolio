# Customer Email Automation

Built with **Google Sheets and Google Apps Script** to manage individual customer emails from the spreadsheet rather than relying on Gmail’s built-in mail merge.

- Select a recipient group and edit its message template from the sheet.
- Check the remaining sending quota, send a test, and send to pending recipients in batches.
- Track status by recipient. A separate 24-hour check looks for any recent message to a pending recipient, regardless of subject, and updates their status before the next batch.

The original workflow stayed subject to Google’s [sending limits](https://support.google.com/mail/answer/12921167) and [Apps Script quotas](https://developers.google.com/apps-script/guides/services/quotas). The public excerpt below illustrates the status check and batch preview without accessing a mailbox, connecting to a spreadsheet, or sending messages. No client records, templates, or identifiers are included.

## Workflow excerpt

```js
const DAY = 24 * 60 * 60 * 1000;
const normalizeEmail = value => String(value ?? '').trim().toLowerCase();
const addresses = value => (String(value ?? '').match(/[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}/gi) ?? []).map(normalizeEmail);
const validEmail = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
const pending = row => String(row.status ?? '').trim().toUpperCase() === 'PENDING';

// Check sample recipient history separately from the batch preview.
function reconcile(rows, messages, {sender, now, limit = 1500}) {
  const owner = normalizeEmail(sender);
  if (!owner || !Number.isFinite(now)) throw new Error('A sender and reference time are required.');
  const recent = new Map();
  for (const [i, message] of messages.entries()) {
    if (i >= limit) throw new Error('History scan limit reached. No statuses were changed.');
    if (!Number.isFinite(message.at) || message.at < now - DAY || message.at > now) continue;
    if (addresses(message.from)[0] !== owner) continue;
    for (const email of addresses(message.to)) {
      recent.set(email, Math.max(message.at, recent.get(email) ?? -Infinity));
    }
  }
  return rows.map(row => {
    const email = normalizeEmail(row.email);
    return pending(row) && recent.has(email)
      ? {...row, status: 'SENT', matchedAt: recent.get(email)}
      : {...row};
  });
}

// Preview a batch from current rows alone. Nothing is sent.
function previewBatch(rows, {batchSize, quota}) {
  const cap = Math.floor(Math.min(batchSize, quota));
  if (!Number.isFinite(cap) || cap <= 0) return [];
  const blocked = new Set(rows.filter(row => !pending(row)).map(row => normalizeEmail(row.email)));
  const seen = new Set();
  const planned = [];
  for (const row of rows) {
    const email = normalizeEmail(row.email);
    if (!pending(row) || !validEmail.test(email) || blocked.has(email) || seen.has(email)) continue;
    seen.add(email);
    planned.push(email);
    if (planned.length === cap) break;
  }
  return planned;
}
```
