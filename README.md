# RMMC-MI Equipment Borrowing System

A browser-first, installable PWA built with Next.js, React, Tailwind CSS, and Supabase. The local development server runs at `http://localhost:5501`.

## Run locally

1. Install Node.js 20.9 or newer.
2. Copy `.env.example` to `.env.local` and set the Supabase project URL and publishable key.
3. This app is configured for the existing RMMC schema (`profiles`, `categories`, `equipment`, `borrowers`, `borrowings`, `borrowing_items`, `activity_logs`, and `notifications`). In the matching Supabase project's SQL Editor, run [`supabase/migrations/202610230002_align_existing_schema.sql`](./supabase/migrations/202610230002_align_existing_schema.sql), [`supabase/migrations/202610230003_auto_create_student_borrowers.sql`](./supabase/migrations/202610230003_auto_create_student_borrowers.sql), and [`supabase/migrations/202610260001_equipment_images_quantity.sql`](./supabase/migrations/202610260001_equipment_images_quantity.sql), in that order. The last migration adds equipment pictures and amounts, creates the public image bucket, and restricts image uploads to administrators. Do **not** run the earlier `202610220001_equipment_borrowing.sql` or `202610230001_equipment_feedback_audit.sql` migrations against this database; those are for the app's original, different schema.
4. Install dependencies and start the app:

   ```powershell
   npm install
   npm run dev
   ```

5. In Supabase, configure the **Magic Link or OTP** email template to show `{{ .Token }}` (instructions below).
6. Open `http://localhost:5501`, create an account, and sign in with the emailed one-time code.

### Configure email-code sign-in

The app requests an email OTP with Supabase `signInWithOtp` and verifies it with `verifyOtp`. The email content is controlled by the Supabase project, not by this app. To send a code instead of a clickable sign-in link:

1. Open **Authentication → Email Templates → Magic Link or OTP** in the Supabase Dashboard.
2. Set the subject to `Login Code: {{ .Token }}`.
3. Replace the template body with HTML that includes `{{ .Token }}` and does not include `{{ .ConfirmationURL }}` or a link/button using it. For example:

   ```html
   <h1>Your sign-in code</h1>
   <p>Enter this code in the app to sign in:</p>
   <p style="font-size:32px;font-weight:bold;letter-spacing:8px">{{ .Token }}</p>
   <p>If you did not request this code, you can ignore this email.</p>
   ```

4. Save the template. The app’s sign-in screen will prompt for the code and verify it. If the email still contains a link, check that the saved template no longer references `{{ .ConfirmationURL }}`.

### Run and debug in VS Code

Open this folder in VS Code and press **F5**. The included launch configuration starts the Next.js development server and opens Chrome at `http://localhost:5501`. You can also run `npm run dev` in the VS Code terminal and open that address in any modern browser.

### Use on a phone

This project is a responsive, installable Progressive Web App (PWA), not a separate native iOS/Android project. Open the deployed HTTPS site on your phone and use **Install app** (or Safari’s **Share → Add to Home Screen**) to add it to your home screen. The installed app uses the same sign-in, equipment, borrowing, reservation, and account actions as the browser version. A development server on your computer can be used for browser testing, but phone installation requires the app to be served over HTTPS.

## First administrator

New accounts receive the `staff` role. After creating and confirming your own account, use the Supabase SQL Editor to promote that account:

```sql
update public.profiles
set role = 'admin'
where id = (
  select id
  from auth.users
  where email = 'your-institutional-email@example.org'
);
```

Only the `admin` role can add, edit, or remove equipment. Equipment changes are recorded in `equipment_audit_log`, and removal is soft-deletion so borrowing history remains intact. `admin` and `property_custodian` roles can review borrowing and reservation requests. Profile role values remain `admin`, `property_custodian`, and `staff`; users cannot change their own role.

## Included workflows

- Supabase email-code sign-in, password-based account registration, password reset, and cookie refresh.
- New signups automatically receive an active `student` borrower row with a generated borrower number; accounts created before the borrower migration may need to be added to `public.borrowers` separately.
- Equipment search, category filters, equipment status, storage location, and administrator-only inventory management with a database-backed change history.
- Shared member/admin feedback discussions with replies and administrator-managed status updates.
- Borrowing and reservation requests linked to the existing `borrowers`, `borrowings`, `borrowing_items`, and `equipment` records, with administrator/property-custodian review and return confirmation.
- Member activity, manager request queues, borrowing history, and CSV report export.
- Installable PWA manifest and an offline page. Private workspace data is never cached by the service worker.

## Notes

- The publishable Supabase key is intended for browser use. Never put a Supabase service-role key in `.env.local` or client code.
- Borrowing and reservation requests require an active borrower row whose email matches the signed-in account.
- If Supabase reports missing tables or functions, ensure the migration ran successfully in the project that matches `.env.local`.
- To create a production build, run `npm run build`, then `npm start`.
