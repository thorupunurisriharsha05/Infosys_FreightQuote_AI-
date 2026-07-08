# Milestone 1 — Freight Quote Portal

## What this milestone is
This milestone delivers an authentication portal for **Freight Quote** — a Streamlit web app with Login, Signup, and Forgot Password pages, session handling via JWT, a customer dashboard, and a separate Admin Dashboard, all running from a single Colab notebook and exposed publicly through ngrok.

## Features built
- **Login** — username/email + password, generic error on failure (never reveals which field was wrong), issues a JWT on success.
- **Signup** — username, email, password, confirm password, security question + answer; enforces a unique username and full field validation.
- **Forgot Password** — two independent recovery routes on one page:
  - **Security Question** route (username → answer → new password)
  - **OTP** route (registered email → 6‑digit code sent via Gmail → new password)
- **JWT session management** — a session token is issued at Login and validated before the Dashboard is shown; Signup and password resets route back to Login instead of minting their own token.
- **Field validation** — no empty-field submissions; email must look like `ab@cd.ef`; passwords need 8+ characters with an uppercase letter, lowercase letter, number, and special symbol.
- **User Dashboard** — welcome banner with the logged-in identity, sample stats, and Logout.
- **Admin Dashboard** — a separate admin login (credentials come from Colab Secrets, not the signup table) that lists every registered username/email — passwords are never shown.

## Tech stack
- **Streamlit** — UI and page routing
- **SQLite** — user store (`freightquote_portal.db`)
- **PyJWT** — signed session tokens
- **bcrypt** — password and security-answer hashing
- **smtplib / email.mime** — OTP delivery over Gmail SMTP
- **pyngrok** — public URL for the Colab-hosted app
- **Plotly** + **streamlit-option-menu** — dashboard visuals and sidebar navigation

## How to run
1. Open `Milestone1.ipynb` in Google Colab.
2. Add the following to Colab Secrets (key icon, left sidebar) and enable notebook access for each:
   - `JWT_SECRET` — any long random string
   - `NGROK_AUTHTOKEN` — from your ngrok dashboard
   - `EMAIL_ADDRESS` / `EMAIL_PASSWORD` — a Gmail address and its 16-character App Password (requires 2‑Step Verification)
   - `ADMIN_EMAIL` / `ADMIN_PASSWORD` *(optional)* — admin login; defaults to `admin@freightquote.com` if not set
3. Run the notebook cells in order: install dependencies → write `app.py` → launch Streamlit + ngrok.
4. Open the printed ngrok URL to use the app.

## Screenshots
_Add screenshots to the `screenshots/` folder inside `Milestone1/` and reference them below._

| Page | Screenshot |
|---|---|
| Login | `screenshots/login.png` |
| Signup | `screenshots/signup.png` |
| Forgot Password — Security Question | `screenshots/forgot_security_question.png` |
| Forgot Password — OTP | `screenshots/forgot_otp.png` |
| OTP Email | `screenshots/otp_email.png` |
| User Dashboard | `screenshots/user_dashboard.png` |
| Admin Dashboard | `screenshots/admin_dashboard.png` |
