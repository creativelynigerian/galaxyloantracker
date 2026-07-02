# Ledger — Personal Loan Tracker

A small site for tracking loans you give out: borrowers apply, you approve
and set terms, and everyone can see the repayment schedule. Built with
plain HTML/CSS/JS, Firebase Auth, and Firestore. Hosted for free on GitHub
Pages.

## What's included

- `index.html` — sign in
- `register.html` — borrower self-registration
- `dashboard.html` — borrower's view of their own loans
- `apply.html` — loan application form
- `admin.html` — your (admin) view: pending approvals + all loans
- `loan-detail.html` — full repayment schedule; approve/reject/mark paid (admin), read-only view (borrower)
- `firestore.rules` — security rules enforcing the admin/borrower split
- `functions/` — **optional** scheduled email reminders (see step 6)

## 1. Create a Firebase project

1. Go to [console.firebase.google.com](https://console.firebase.google.com) → **Add project**.
2. Once created, click the **`</>`** (web) icon to register a web app. Name it anything.
3. Copy the `firebaseConfig` object it gives you.
4. Paste those values into `js/firebase-config.js` in this project.

## 2. Turn on Authentication

1. In the Firebase Console, go to **Build → Authentication → Get started**.
2. Enable the **Email/Password** sign-in method.

## 3. Turn on Firestore

1. Go to **Build → Firestore Database → Create database**.
2. Start in **production mode** (we'll set our own rules).
3. Once created, go to the **Rules** tab, delete the default contents, and
   paste in everything from `firestore.rules` in this project. Click **Publish**.

## 4. Make yourself the admin

1. Open your deployed site (or run it locally — see step 5) and **register**
   an account for yourself using `register.html`.
2. In the Firebase Console, go to **Firestore Database → Data**, open the
   `users` collection, find your document (matches your account's UID),
   and change the `role` field from `borrower` to `admin`.
3. Sign out and back in — you'll now land on `admin.html`.

Everyone else who registers stays a `borrower` automatically, and the
security rules stop anyone from setting their own role to `admin` — that
one-time console edit is intentional and is the only way to grant it.

## 5. Run it locally (optional, for testing)

No build step is needed — it's static files. Easiest option:

```bash
# from the project folder
python3 -m http.server 8000
# then open http://localhost:8000
```

Or use the VS Code "Live Server" extension, or any static file server.

## 6. Deploy with GitHub Pages

1. Create a new GitHub repository and push this folder to it:
   ```bash
   git init
   git add .
   git commit -m "Initial commit"
   git branch -M main
   git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO.git
   git push -u origin main
   ```
2. In the repo on GitHub: **Settings → Pages**.
3. Under **Source**, choose **Deploy from a branch**, pick `main` and `/ (root)`, then **Save**.
4. After a minute or two, your site will be live at
   `https://YOUR_USERNAME.github.io/YOUR_REPO/`.

⚠️ **Note:** `js/firebase-config.js` contains your Firebase project's public
web config. This is normal and fine to have in a public repo — it's not a
secret key, it just tells the browser which project to talk to. Access
control is enforced by `firestore.rules`, not by hiding this file. Also add
your GitHub Pages domain to **Firebase Console → Authentication → Settings
→ Authorized domains** if it isn't there automatically.

## 7. (Optional) Turn on email reminders

The site already shows **overdue** / due-date status for free, right in
the UI — no setup needed. If you also want actual emails sent out:

1. Upgrade your Firebase project to the **Blaze** (pay-as-you-go) plan.
   Scheduled functions require it, but a handful of emails a day costs
   fractions of a cent.
2. Sign up for [SendGrid](https://sendgrid.com) (or swap in another
   provider) and verify a sender email address.
3. In the `functions/` folder:
   ```bash
   cd functions
   npm install
   firebase functions:secrets:set SENDGRID_API_KEY
   firebase deploy --only functions
   ```
4. Edit `FROM_EMAIL` in `functions/index.js` to your verified sender address.

SMS reminders work the same way, swapping SendGrid for a provider like
Twilio inside the same scheduled function.

## How the numbers work

When you approve a loan, the site builds an **equal-installment schedule**
using simple annual interest:

```
totalInterest = principal × (rate / 100) × (months / 12)
totalDue      = principal + totalInterest
installment   = totalDue / months
```

Each installment is due one month apart, starting from the approval date.

## Data model (Firestore)

**`users/{uid}`**
```
{ name, email, role: "borrower" | "admin", createdAt }
```

**`loans/{loanId}`**
```
{
  borrowerId, borrowerName, borrowerEmail,
  amount, termMonths, interestRate, purpose,
  status: "pending" | "active" | "rejected" | "paid",
  schedule: [{ index, dueDate, amount, paid, paidDate }],
  totalDue, createdAt, approvedAt
}
```

"Overdue" isn't a stored status — it's computed on the fly whenever a
loan is `active` and has an unpaid installment past its due date.
