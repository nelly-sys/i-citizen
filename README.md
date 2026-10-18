# i-citizen

A prototype for citizens to submit service requests and track them. The front end is React + TypeScript + Vite, the backend is Supabase (Postgres, Auth, Storage), and it's hosted on Netlify. All the access control lives in the database, using Row Level Security and SQL functions that check scope. I didn't rely on hiding buttons in the UI.

## What works

- **Citizens** can register (Supabase sends the verification email), sign in and out, and submit requests with an optional private image (JPEG, PNG or WebP, up to 5 MB). Each request gets a reference number like `CCR-2026-000123` from a database sequence. They can see their own requests with a status filter and pagination, open a request to see its timeline (built from `request_events`), and send follow-up messages.
- **Staff** (officer, supervisor, admin) get a dashboard with metrics from the database for the last 7, 30 or 90 days, plus requests by category and a searchable, filterable table. They can change a request's status (transitions are validated and a resolution note is required), post public updates, add internal notes that citizens never see, and set the department and priority.
- The database writes in-app notification rows and an audit log entry when a request is submitted, its status changes, or it gets assigned.

## What's not built yet

- A notifications bell/UI and email delivery
- Admin screens for departments, categories, users and audit logs. For now, manage these in the Supabase table editor or call `admin_set_user`
- CSV export and custom date ranges
- Service targets and overdue counts
- Reopening requests
- Public tracking by reference number
- Automated tests

Please don't use this with real citizen data until those are done, someone has done a security review, and a data-protection assessment has been completed (Kenya Data Protection Act).

## 1. Set up Supabase

1. Create a project at supabase.com.
2. Open the SQL Editor and run `supabase/migrations/001_init.sql`, then `supabase/seed.sql`. The seed adds the departments and categories.
3. Go to Authentication → URL Configuration. Set the Site URL to your Netlify URL, and add `http://localhost:5173/**` and `https://YOUR-SITE.netlify.app/**` to the Redirect URLs.
4. Go to Authentication → Providers → Email and leave "Confirm email" switched on. Under Rate Limits you can keep the defaults or make them stricter.
5. Go to Project Settings → API and copy the Project URL and the **anon** key. Never put the `service_role` key in the front end or in Netlify's build variables.

## 2. Run it locally

```
npm install
cp .env.example .env   # fill in the two values
npm run dev            # http://localhost:5173
npm run build          # type-check and production build
```

## 3. Push to GitHub and deploy on Netlify

```
git init && git add . && git commit -m "i-citizen"
git remote add origin https://github.com/YOU/i-citizen.git && git push -u origin main
```

In Netlify, click Add new site → Import from Git and pick the repo. `netlify.toml` already has the build command, the `dist` publish folder, the SPA redirects and the security headers, so you don't need to set those. Add the environment variables `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY`, then deploy. Netlify gives you HTTPS automatically. Once you know the final site URL, go back and update the redirect URLs in Supabase.

## 4. Make the first admin

Register in the app like a normal user and verify your email. Then run this in the SQL editor:

```sql
update profiles set role = 'admin' where id = (select id from auth.users where email = 'you@example.com');
```

You can create officers and supervisors the same way, for example:

```sql
update profiles set role='officer', department_id=(select id from departments where name='Water Services') where id = ...;
```

Clients can't set roles themselves. Sign-up always makes a citizen, and `profiles.role` can't be updated through the API.

## Roles

| | Citizen | Officer | Supervisor | Admin |
|---|---|---|---|---|
| Own requests, messages | yes | – | – | – |
| Requests in own department / assigned | – | yes | yes (department) | all |
| Change status, notes, public updates | – | yes, within scope | yes | yes |
| Assign department / priority | – | – | yes | yes |
| Internal notes | never | yes | yes | yes |
| Categories, departments, user roles, audit logs | – | – | – | yes |

## How the metrics are defined

- **Open** means Pending plus In Progress.
- **Unassigned** means not resolved and no department set.
- **Average resolution time** only counts resolved requests where `resolved_at >= submitted_at`.
- Archived requests are left out.
- Every metric runs under the caller's RLS scope, so people only see numbers for what they're allowed to see.

## Security notes

RLS is on for every table. Staff writes go through `security definer` functions that check scope and allowed transitions. Images sit in a private bucket and are shown through signed URLs that expire after 5 minutes, and storage enforces the MIME type and size limit on the server side. Audit logs can only be read by admins. The anon key is public by design. Before going live, log in with a second citizen account and test that the policies really stop it from seeing the first one's requests.

## Free-tier limits (check the current terms)

Free Supabase projects pause after a stretch of inactivity, have limits on database and storage size, and have a low built-in email rate limit, so set up your own SMTP for anything real. Netlify's free plan has monthly limits on build minutes and bandwidth.

## Backups

Free Supabase doesn't include managed point-in-time restore. Export regularly with `supabase db dump` or `pg_dump` using your connection string, and back up the storage bucket separately.