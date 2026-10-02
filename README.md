# ClassConnect

A secure, responsive school communication app with teacher-managed classrooms, private one-to-one chats, unique `@usernames`, per-user pinned conversations, account settings, and a metadata-only administrator panel. Student classroom text, image, and file permissions are independently managed per class and default to off. The project uses **Next.js App Router, TypeScript, Prisma, SQLite, bcrypt, and private local file storage**.

## What is included

- Email/password registration and login by email or username, globally unique normalized usernames, bcrypt password hashes, HTTP-only session cookies, logout, and a 30-day “Remember me” option.
- Public registration always creates a **student**. Bootstrap the first administrator from the server CLI with `create:admin`; thereafter admins can manage student, teacher, and admin roles in the panel. Teacher privileges are never self-selected at sign-up.
- Teacher-only classroom creation, editing, deletion, student access management, and independent per-class student text, image, and file permissions (each defaults off).
- Students retain read-only classroom feeds until their teacher enables the corresponding permission; any signed-in account can add another account by exact username and start a private one-to-one text conversation.
- Admin account directory with role/status controls, permanent deletion, and audit history. Admins see account metadata and classroom memberships only—not private message bodies or files. Passwords are bcrypt hashes and cannot be read.
- Self-service profile name, unique username, private profile photo, and password updates; account-specific Light, Dark, Instagram, WhatsApp, or Snapchat themes. Email changes are held for verification and are not exposed as an unverified edit.
- Per-user pin/unpin for classroom chats and direct conversations; pinned items sort to the top of their lists.
- Server-side role and classroom authorization on every private API route; files are stored outside `public/` and require a valid session plus classroom access.
- Real-time-style updates via three-second polling, chronological messages, timestamps, upload progress, confirmation prompts, loading/error/empty states, and responsive desktop/mobile layouts.
- SQLite schema, indexes, foreign keys, and a checked-in initial Prisma migration.

Usernames are normalized to lowercase, must be 3–20 characters using letters, numbers, or underscores, and are protected by a database uniqueness constraint. Login accepts either username or email. Adding an account by username creates a reciprocal contact and private direct conversation; classroom student posting remains disabled.

## Requirements

- Node.js 20.9 or later (Node 20 LTS recommended)
- npm

## 1. Install dependencies

From the project root:

```bash
npm ci
```

## 2. Configure environment variables

Create your local environment file:

```bash
cp .env.example .env
```

The default `.env.example` is ready to use:

```env
DATABASE_URL="file:./dev.db"
UPLOAD_DIR="./data/uploads"
```

Prisma resolves the SQLite path relative to `prisma/schema.prisma`, so this creates `prisma/dev.db`. The uploads folder is relative to the project root and is intentionally outside the public web directory. Keep `.env`, the database, and `data/uploads/` private and backed up.

## 3. Create the database

Apply the checked-in migration and generate the Prisma client:

```bash
npm run db:migrate
npx prisma generate
```

The migrations create users (with a unique username), sessions, classrooms (including three default-off student permissions), classroom memberships, class/direct messages, contacts, per-user pins, administrator audit logs, profile/theme fields, and indexes/cascading relationships. The second migration also backfills a unique handle for accounts created by an earlier version. To inspect local data during development:

```bash
npm run db:studio
```

## 4. Bootstrap the first administrator (server CLI only)

Create the first admin after migrations are applied. The CLI is intentionally not exposed as a public registration option:

```bash
ADMIN_NAME="School Administrator" \
ADMIN_USERNAME="school_admin" \
ADMIN_EMAIL="admin@school.edu" \
ADMIN_PASSWORD="use-a-long-unique-password" \
npm run create:admin
```

The command validates the fields, hashes the password with bcrypt, and creates or promotes the specified account to `ADMIN`. Treat environment variables and shell access as secrets. Admins may subsequently assign roles, suspend/restore accounts, or permanently delete accounts from **Admin panel**. The admin panel never exposes passwords, private message bodies, or files.

PowerShell equivalent:

```powershell
$env:ADMIN_NAME = "School Administrator"
$env:ADMIN_USERNAME = "school_admin"
$env:ADMIN_EMAIL = "admin@school.edu"
$env:ADMIN_PASSWORD = "use-a-long-unique-password"
npm run create:admin
```

To provision a teacher from a trusted server shell (optional; admins can also change roles in the panel):

```bash
TEACHER_NAME="Jordan Lee" TEACHER_USERNAME="jordan_lee" \
TEACHER_EMAIL="jordan@school.edu" TEACHER_PASSWORD="use-a-long-unique-password" \
npm run create:teacher
```

Both provisioning scripts hash passwords before saving. Protect server-shell access and environment variables; running a provisioning command again for the same email resets the password and role.

## 5. Start the development server

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000). Run a production build with `npm run build`, then serve it with `npm start`.

## 6. Test teacher functionality

1. Sign in with a teacher account provisioned by CLI or assigned by an administrator.
2. Choose **Create a class** and add a title and optional description.
3. Open the class, then use **Access** to add a student by the email address they registered with.
4. Open the class settings and independently enable student text, image, and file permissions as needed (all are off by default). Save changes; the server enforces the settings even if a student bypasses the UI.
5. Send a text message; use the paperclip to upload a JPG, PNG, WEBP, or PDF (maximum 10 MB).
6. Edit/delete the classroom, remove students, and delete the teacher’s own messages/files from the class controls.

## 7. Test student functionality

1. Sign out.
2. Choose **Create an account**. Register a display name, a unique username, email, and password; public sign-up creates a student account only.
3. Sign in by username or email and confirm the teacher’s class is not visible yet.
4. Sign out, return to the teacher account, and add the student’s exact registered email through **Access**.
5. Sign in as the student again. The class and its messages/files should now be visible. Students can open/download files but cannot see class-management controls; classroom text, images, and files are allowed only when the corresponding class toggles are on. Confirm a disabled action is denied by the backend.
6. Go to **People**, add another account by its exact `@username`, and exchange direct messages. Either user can pin the conversation; pinned items rise to the top of the list.
7. Visit **Profile** to update your display name, unique username (requires your current password), or profile photo. Visit **Settings** to change your password and pick a theme; email changes are not available without verification.

## 8. Test admin functionality

1. Sign in with the CLI-provisioned admin. The Admin panel is the admin landing screen.
2. Search/filter accounts, change a role, suspend and restore an account, and review the audit history. Confirm the UI asks for confirmation before permanent deletion.
3. Confirm the account directory shows metadata and class membership counts only. It provides no access to class/DM message bodies or uploaded files. Never use real production credentials or data in a shared demo environment.

## Session behavior

Sessions use random, high-entropy bearer tokens; only SHA-256 token hashes are stored in the database. Cookies are `HttpOnly`, `SameSite=Lax`, and `Secure` in production. With **Remember me** enabled, the cookie and server session last 30 days. With it disabled, the cookie is a browser-session cookie and the server session expires after 12 hours. A browser configured to restore session cookies may preserve session cookies when reopening; that behavior is controlled by the browser.

## API overview

| Route | Purpose | Authorization |
| --- | --- | --- |
| `POST /api/auth/register` | Create a student account with a unique username | Public; role is fixed to `STUDENT` |
| `POST /api/auth/login` / `POST /api/auth/logout` | Sign in by username/email and sign out | Public / current session |
| `GET /api/auth/session` | Check the current session | Current session |
| `PATCH /api/profile` | Update display name/username and theme | Current session (current password for username changes) |
| `POST, DELETE /api/profile/photo` / `GET /api/avatars/:userId` | Upload/remove a private profile photo / serve it to signed-in users | Owner / current session |
| `POST /api/profile/password` | Change password | Current session and current password |
| `GET /api/admin/accounts` | List account metadata, class counts, and audit history | Admin only; never returns messages/files/passwords |
| `PATCH, DELETE /api/admin/accounts/:userId` | Change role/status or permanently remove account data | Admin only; destructive action is audited |
| `GET, POST /api/contacts` | List contacts / add an account by username and open a direct conversation | Current session |
| `GET, POST /api/chats` | List accessible classes / create class | Session / teacher |
| `PUT /api/chats/:chatId/pin` | Pin/unpin a classroom for the current user | Classroom member or owner |
| `GET, PATCH, DELETE /api/chats/:chatId` | Read/manage one class | Member or owner / owner teacher for mutations |
| `GET, POST /api/chats/:chatId/messages` | Read/post messages | Member or owner / owner teacher for posts |
| `POST /api/chats/:chatId/files` | Upload supported file | Teacher owner or student when the class allows that image/file type |
| `GET /api/files/:messageId` | Open/download private file | Class member or owner |
| `GET, POST /api/chats/:chatId/members` | List/add students | Owner teacher only |
| `DELETE /api/chats/:chatId/members/:userId` | Remove student access | Owner teacher only |
| `DELETE /api/messages/:messageId` | Delete a teacher’s own message/file | Owner teacher and original sender |
| `GET, POST /api/direct-conversations/:id/messages` | Read/send private one-to-one messages | Conversation participant |
| `PUT /api/direct-conversations/:id/pin` | Pin/unpin a direct chat for the current user | Conversation participant |

## Security and deployment notes

- All user-facing text is rendered as text (not injected HTML); inputs have server-side length and shape validation.
- Login/registration attempts are throttled per IP/email in the application process. For a multi-instance deployment, move throttling to a shared store or reverse proxy.
- Upload extensions, detected file signatures, content types, and sizes are checked server-side. Only generated UUID filenames are stored; uploads are not exposed as public static files.
- The local default is SQLite plus local disk, which is appropriate for development and a single persistent server. For horizontally scaled/serverless production, use PostgreSQL, migrate the Prisma datasource and database, move uploads to private object storage, and use a shared rate limiter. Persist and back up both the database and upload directory.
- Terminate HTTPS in production so secure session cookies are sent. Use strong operational secrets, regular dependency updates, backups, and monitoring.

## Project layout

```text
app/                   Next.js pages, styles, and API routes
components/            Auth, dashboard, classroom/people chats, settings, admin, and profile UI
lib/                   Prisma client, auth/session, access helpers, file utilities, types
prisma/schema.prisma   Database models, role/status, themes, class permissions, and indexes
prisma/migrations/     SQLite migrations
scripts/               Secure administrator and teacher provisioning commands
data/uploads/          Private uploaded files (created at runtime; git-ignored)
```
