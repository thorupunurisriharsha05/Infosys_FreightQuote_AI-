%%writefile app.py
import os, re, sqlite3, jwt, bcrypt, datetime, time, secrets, smtplib, streamlit as st
import plotly.graph_objects as go
from streamlit_option_menu import option_menu
from email.utils import formatdate, make_msgid
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart



os.makedirs(".streamlit", exist_ok=True)
with open(".streamlit/config.toml", "w") as f:
    f.write('[theme]\nbase="light"\nprimaryColor="#3aa0ff"\nbackgroundColor="#ffffff"\n'
            'secondaryBackgroundColor="#eaf4fd"\ntextColor="#123a5e"\n')

st.set_page_config(page_title="Freight Quote", page_icon="⚡", layout="wide", initial_sidebar_state="expanded")



COLORS = {
    "bg_main": "#ffffff", "bg_sidebar": "#eaf4fd", "bg_card": "#ffffff", "bg_card_alt": "#d6ecff",
    "text_main": "#123a5e", "text_heading": "#0b2942", "text_muted": "#5b7a99",
    "accent": "#3aa0ff", "accent_hover": "#1e86e6", "accent_text": "#ffffff",
    "border": "#0b2942", "border_light": "#bfe2ff", "success": "#34d399", "danger": "#f87171"
}



JWT_SECRET = os.environ.get("JWT_SECRET")
SENDER_EMAIL = os.environ.get("EMAIL_ADDRESS")
EMAIL_PASSWORD = os.environ.get("EMAIL_PASSWORD")
ADMIN_EMAIL = os.environ.get("ADMIN_EMAIL", "admin@freightquote.com")
ADMIN_PASSWORD = os.environ.get("ADMIN_PASSWORD")
OTP_EXPIRY_MINUTES = 5

if not JWT_SECRET:
    st.error("❌ JWT_SECRET is not set. Add it to Colab Secrets (see README Step 6).")
    st.stop()



st.markdown(f"""
<style>
    @import url('https://fonts.googleapis.com/css2?family=Poppins:wght@400;500;600;700&family=Inter:wght@300;400;500;600&display=swap');
    html, body, .stApp {{ background: {COLORS['bg_main']} !important; font-family: 'Inter', sans-serif !important; color: {COLORS['text_main']} !important; }}

    footer, div[data-testid="stDecoration"] {{ visibility: hidden !important; display: none !important; }}
    header {{ background: transparent !important; z-index: 999999 !important; }}

    button[kind="header"], div[data-testid="stSidebarCollapsedControl"] button {{
        visibility: visible !important; display: flex !important; opacity: 1 !important;
        background-color: {COLORS['accent']} !important; border: 2px solid {COLORS['border']} !important;
        border-radius: 8px !important; padding: 6px !important; margin: 8px !important;
        box-shadow: 3px 3px 0px {COLORS['border']} !important;
    }}
    button[kind="header"] svg, div[data-testid="stSidebarCollapsedControl"] svg {{
        fill: #ffffff !important; color: #ffffff !important; stroke: #ffffff !important;
    }}

    .block-container {{ padding: 2rem 2.5rem !important; max-width: 1200px; }}
    h1, h2, h3, h4 {{ font-family: 'Poppins', sans-serif !important; color: {COLORS['text_heading']} !important; }}
    label p {{ font-weight: 600 !important; color: {COLORS['text_heading']} !important; }}

    div[data-baseweb="base-input"], div[data-baseweb="select"] > div {{ background-color: transparent !important; border: none !important; }}
    div[data-baseweb="input"], div[data-baseweb="select"] {{ background-color: {COLORS['bg_card']} !important; border: 2px solid {COLORS['border']} !important; border-radius: 10px !important; }}
    div[data-baseweb="input"]:focus-within {{ border-color: {COLORS['accent']} !important; box-shadow: 4px 4px 0px {COLORS['border']} !important; }}
    input, div[data-baseweb="select"] span {{ color: {COLORS['text_main']} !important; -webkit-text-fill-color: {COLORS['text_main']} !important; }}

    div[data-testid="stButton"] button {{
        background-color: {COLORS['accent']} !important; color: {COLORS['accent_text']} !important;
        border: 2px solid {COLORS['border']} !important; border-radius: 10px !important;
        font-family: 'Inter', sans-serif !important; font-weight: 700 !important; font-size: 14px !important;
        height: 48px !important; min-height: 48px !important; white-space: nowrap !important;
        display: flex !important; align-items: center !important; justify-content: center !important;
        padding: 0px 16px !important; box-shadow: 4px 4px 0px {COLORS['border']} !important; width: 100%; transition: all 0.2s ease !important;
    }}
    div[data-testid="stButton"] button:hover {{
        background-color: {COLORS['accent_hover']} !important; transform: translate(-2px, -2px) !important;
        box-shadow: 6px 6px 0px {COLORS['border']} !important;
    }}
    section[data-testid="stSidebar"] {{ background: {COLORS['bg_sidebar']} !important; border-right: 2px solid {COLORS['border']} !important; }}
    .pn-card {{ background: {COLORS['bg_card']}; border: 2px solid {COLORS['border']}; border-radius: 14px; padding: 24px; box-shadow: 4px 4px 0px {COLORS['border_light']}; }}
    div[data-testid="stVerticalBlockBorderWrapper"] {{
        background-color: {COLORS['bg_card']} !important; border: 2px solid {COLORS['border']} !important;
        border-radius: 16px !important; padding: 24px !important; box-shadow: 6px 6px 0px {COLORS['border_light']} !important;
    }}
</style>
""", unsafe_allow_html=True)



def get_db(): return sqlite3.connect("freightquote_portal.db", check_same_thread=False)
def hash_txt(t): return bcrypt.hashpw(t.encode(), bcrypt.gensalt()).decode()
def check_txt(t, h): return bcrypt.checkpw(t.encode(), h.encode()) if h else False

with get_db() as conn:
    conn.execute("""CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT UNIQUE, email TEXT UNIQUE,
        password_hash TEXT, security_question TEXT, security_answer_hash TEXT)""")

SECURITY_QUESTIONS = [
    "What is your pet's name?",
    "What is your mother's maiden name?",
    "What is your favourite city?",
]



EMAIL_RE = re.compile(r'^[A-Za-z]{2,}[A-Za-z0-9._%+\-]*@[A-Za-z]{2,}[A-Za-z0-9\-]*\.[A-Za-z]{2,}$')

def valid_email(e): return bool(EMAIL_RE.match(e or ""))

def valid_password(p):
    if not p or len(p) < 8: return False
    if not re.search(r'[A-Z]', p): return False
    if not re.search(r'[a-z]', p): return False
    if not re.search(r'[0-9]', p): return False
    if not re.search(r'[^A-Za-z0-9]', p): return False
    return True

PASSWORD_HELP = "Min 8 chars, 1 uppercase, 1 lowercase, 1 number, 1 special symbol."




def make_jwt(email):
    return jwt.encode({"email": email, "exp": datetime.datetime.utcnow() + datetime.timedelta(hours=2)},
                       JWT_SECRET, algorithm="HS256")

def verify_jwt(token):
    try: return jwt.decode(token, JWT_SECRET, algorithms=["HS256"])
    except Exception: return None

def make_otp_token(email, otp):
    payload = {"sub": email, "otp_hash": hash_txt(otp), "type": "password_reset_otp",
               "iat": datetime.datetime.utcnow(), "exp": datetime.datetime.utcnow() + datetime.timedelta(minutes=OTP_EXPIRY_MINUTES)}
    return jwt.encode(payload, JWT_SECRET, algorithm="HS256")

def verify_otp_token(token, input_otp, email):
    try:
        payload = jwt.decode(token, JWT_SECRET, algorithms=["HS256"])
        if payload.get("sub") != email or payload.get("type") != "password_reset_otp":
            return False, "Security token mismatch."
        if check_txt(input_otp, payload["otp_hash"]): return True, "Valid"
        return False, "Invalid 6-digit OTP code."
    except jwt.ExpiredSignatureError:
        return False, f"⚠️ This OTP code expired after {OTP_EXPIRY_MINUTES} minutes. Please request a new one."
    except Exception:
        return False, "Invalid or corrupted verification token."

def generate_otp(): return f"{secrets.randbelow(900000) + 100000}"

def send_otp_email(to_email, otp):
    if not SENDER_EMAIL or not EMAIL_PASSWORD:
        return False, "EMAIL_ADDRESS / EMAIL_PASSWORD not configured in Colab Secrets."
    msg = MIMEMultipart('alternative')
    msg['From'] = f"Freight Quote Support <{SENDER_EMAIL}>"
    msg['To'] = to_email
    msg['Subject'] = "Freight Quote - Verification Code"
    msg['Date'] = formatdate(localtime=True)
    msg['Message-ID'] = make_msgid()
    msg['Reply-To'] = SENDER_EMAIL

    text_body = (f"Your verification code for Freight Quote is: {otp}\n"
                 f"This code will expire in {OTP_EXPIRY_MINUTES} minutes.\n"
                 f"If you did not request this code, please ignore this email.")

    html_body = f"""
    <html><body style="font-family:Arial,sans-serif;background:#eaf4fd;padding:20px;">
      <div style="max-width:480px;margin:0 auto;background:#ffffff;border:2px solid #0b2942;border-radius:12px;padding:30px;text-align:center;">
        <div style="color:#0b2942;font-size:20px;font-weight:bold;margin-bottom:15px;">Freight Quote Verification</div>
        <div style="color:#123a5e;font-size:15px;margin-bottom:20px;">We received a request to reset the password for <b>{to_email}</b>. Use the code below:</div>
        <div style="background:#3aa0ff;color:#ffffff;font-size:28px;font-weight:bold;letter-spacing:5px;padding:15px 20px;border:2px solid #0b2942;border-radius:8px;display:inline-block;margin:10px 0;">{otp}</div>
        <div style="color:#123a5e;font-size:14px;margin-top:15px;">This code expires in <b>{OTP_EXPIRY_MINUTES} minutes</b>.</div>
      </div>
    </body></html>"""

    msg.attach(MIMEText(text_body, 'plain'))
    msg.attach(MIMEText(html_body, 'html'))
    try:
        s = smtplib.SMTP('smtp.gmail.com', 587)
        s.starttls()
        s.login(SENDER_EMAIL, EMAIL_PASSWORD)
        s.sendmail(SENDER_EMAIL, to_email, msg.as_string())
        s.quit()
        return True, "Email sent successfully!"
    except Exception as e:
        return False, f"SMTP Error: {str(e)}"




for k, v in [("token", None), ("page", "Login"), ("reset_email", None), ("reset_mode", None),
             ("otp_token", None), ("fp_verified", False)]:
    if k not in st.session_state: st.session_state[k] = v

def navigate(p): st.session_state.page = p; st.rerun()

def auth_header(title, sub="Freight Quote Portal"):
    st.markdown(f"""S
    <div style="text-align:center;padding:1.5rem 0 1rem;">
        <div style="font-size:40px;margin-bottom:10px;">⚡</div>
        <h1 style="font-size:2rem !important;margin:0;">Freight Quote</h1>
        <p style="color:{COLORS['text_muted']};font-size:14px;margin:4px 0 0;">{sub}</p>
    </div>
    <div style="text-align:center;margin-bottom:1.5rem;"><span style="font-size:1.1rem;font-weight:700;color:{COLORS['text_heading']};">{title}</span></div>
    """, unsafe_allow_html=True)



if not st.session_state.token:
    if st.session_state.page not in ["Login", "Signup", "Forgot"]:
        st.session_state.page = "Login"

    _, mid, _ = st.columns([1, 1.45, 1])
    with mid:

        # ---------------- LOGIN ----------------
        if st.session_state.page == "Login":
            auth_header("Sign in to your account")
            email = st.text_input("Email address", placeholder="you@freightquote.com").lower().strip()
            pwd = st.text_input("Password", type="password", placeholder="••••••••")
            st.markdown("<br>", unsafe_allow_html=True)

            col_l, col_c, col_r = st.columns([1, 1.15, 1.3])
            if col_l.button("Sign In →", use_container_width=True):
                if not email or not pwd:
                    st.error("⚠️ Both fields are required.")
                else:
                    with get_db() as c:
                        r = c.execute("SELECT password_hash FROM users WHERE email=?", (email,)).fetchone()
                    if r and check_txt(pwd, r[0]):
                        st.session_state.token = make_jwt(email); navigate("Dashboard")
                    else:
                        st.error("❌ Invalid credentials.")  # generic — no hint which field failed
            if col_c.button("Create Account", use_container_width=True): navigate("Signup")
            if col_r.button("Forgot Password", use_container_width=True): navigate("Forgot")

        # ---------------- SIGNUP ----------------
        elif st.session_state.page == "Signup":
            auth_header("Create an account", "Join Freight Quote today")
            uname = st.text_input("Username", placeholder="Jane Doe")
            email = st.text_input("Email address", placeholder="you@freightquote.com").lower().strip()
            pwd = st.text_input("Password", type="password", placeholder="Min. 8 characters", help=PASSWORD_HELP)
            confirm_pwd = st.text_input("Confirm password", type="password", placeholder="Re-enter password")
            sq = st.selectbox("Security Question", SECURITY_QUESTIONS)
            sa = st.text_input("Your answer", placeholder="Security answer")
            st.markdown("<br>", unsafe_allow_html=True)

            if st.button("Create Account & Login →", use_container_width=True):
                if not uname or not email or not pwd or not confirm_pwd or not sa:
                    st.error("⚠️ Please fill in all fields.")
                elif not valid_email(email):
                    st.error("❌ Please enter a valid email address (e.g. ab@cd.ef).")
                elif not valid_password(pwd):
                    st.error(f"❌ Weak password. {PASSWORD_HELP}")
                elif pwd != confirm_pwd:
                    st.error("❌ Passwords do not match.")
                else:
                    try:
                        with get_db() as c:
                            existing = c.execute("SELECT 1 FROM users WHERE username=?", (uname,)).fetchone()
                            if existing:
                                st.error("❌ That username is already taken.")
                                st.stop()
                            c.execute("INSERT INTO users VALUES (NULL, ?, ?, ?, ?, ?)",
                                      (uname, email, hash_txt(pwd), sq, hash_txt(sa.lower().strip())))
                        st.success("✅ Account created! Please sign in.")
                        time.sleep(1)
                        navigate("Login")
                    except sqlite3.IntegrityError:
                        st.error("❌ Username or email already registered.")

            st.markdown("<br>", unsafe_allow_html=True)
            if st.button("← Back to Sign In", use_container_width=True): navigate("Login")

        # ---------------- FORGOT PASSWORD (Security Question + OTP) ----------------
        elif st.session_state.page == "Forgot":
            auth_header("Reset your password", "Choose your verification method")

            if not st.session_state.reset_mode:
                st.markdown("<h4 style='text-align:center;'>Security Question route</h4>", unsafe_allow_html=True)
                uname_in = st.text_input("Username", key="sq_username")
                col_sq, _ = st.columns([1, 1])
                if col_sq.button("Continue with Security Question", use_container_width=True):
                    if not uname_in:
                        st.error("⚠️ Enter your username.")
                    else:
                        with get_db() as c:
                            r = c.execute("SELECT security_question, email FROM users WHERE username=?", (uname_in,)).fetchone()
                        if r:
                            st.session_state.reset_mode = "sq"
                            st.session_state.sq_prompt = r[0]
                            st.session_state.reset_email = r[1]
                            st.session_state.sq_username = uname_in
                            st.rerun()
                        else:
                            st.error("❌ Username not found.")

                st.markdown("<hr>", unsafe_allow_html=True)
                st.markdown("<h4 style='text-align:center;'>OTP (email) route</h4>", unsafe_allow_html=True)
                otp_email_in = st.text_input("Registered email address", key="otp_email_input").lower().strip()
                col_otp, _ = st.columns([1, 1])
                if col_otp.button("Send OTP to Email", use_container_width=True):
                    if not otp_email_in:
                        st.error("⚠️ Enter your registered email.")
                    else:
                        with get_db() as c:
                            r = c.execute("SELECT 1 FROM users WHERE email=?", (otp_email_in,)).fetchone()
                        if not r:
                            st.error("❌ Email not registered.")
                        else:
                            otp = generate_otp()
                            with st.spinner("Sending OTP..."):
                                ok, msg = send_otp_email(otp_email_in, otp)
                            if ok:
                                st.session_state.reset_mode = "otp"
                                st.session_state.reset_email = otp_email_in
                                st.session_state.otp_token = make_otp_token(otp_email_in, otp)
                                st.session_state.fp_verified = False
                                st.success("✅ OTP sent! Check your inbox.")
                                time.sleep(1)
                                st.rerun()
                            else:
                                st.error(f"❌ {msg}")

            elif st.session_state.reset_mode == "sq":
                st.info(f"❓ **Security Question:** {st.session_state.sq_prompt}")
                ans = st.text_input("Your answer").lower().strip()
                npw = st.text_input("New password", type="password", help=PASSWORD_HELP)
                confirm_npw = st.text_input("Confirm new password", type="password")
                st.markdown("<br>", unsafe_allow_html=True)
                if st.button("Reset Password →", use_container_width=True):
                    if not ans or not npw or not confirm_npw:
                        st.error("⚠️ All fields are required.")
                    elif not valid_password(npw):
                        st.error(f"❌ Weak password. {PASSWORD_HELP}")
                    elif npw != confirm_npw:
                        st.error("❌ Passwords do not match.")
                    else:
                        with get_db() as c:
                            r = c.execute("SELECT security_answer_hash FROM users WHERE email=?",
                                          (st.session_state.reset_email,)).fetchone()
                        if r and check_txt(ans, r[0]):
                            with get_db() as c:
                                c.execute("UPDATE users SET password_hash=? WHERE email=?",
                                          (hash_txt(npw), st.session_state.reset_email))
                            st.success("✅ Password updated! Please sign in.")
                            time.sleep(1)
                            st.session_state.reset_mode = None
                            st.session_state.reset_email = None
                            navigate("Login")
                        else:
                            st.error("❌ Incorrect security answer.")

            elif st.session_state.reset_mode == "otp":
                if not st.session_state.fp_verified:
                    st.info(f"📧 Code sent to **{st.session_state.reset_email}** (valid {OTP_EXPIRY_MINUTES} min).")
                    otp_input = st.text_input("6-digit OTP", max_chars=6)
                    if st.button("Verify OTP →", use_container_width=True):
                        if not otp_input or len(otp_input) != 6:
                            st.error("⚠️ Enter the 6-digit code.")
                        else:
                            ok, msg = verify_otp_token(st.session_state.otp_token, otp_input, st.session_state.reset_email)
                            if ok:
                                st.session_state.fp_verified = True
                                st.success("✅ OTP verified!")
                                time.sleep(1)
                                st.rerun()
                            else:
                                st.error(f"❌ {msg}")
                else:
                    npw = st.text_input("New password", type="password", help=PASSWORD_HELP)
                    confirm_npw = st.text_input("Confirm new password", type="password")
                    if st.button("Update Password →", use_container_width=True):
                        if not npw or not confirm_npw:
                            st.error("⚠️ All fields are required.")
                        elif not valid_password(npw):
                            st.error(f"❌ Weak password. {PASSWORD_HELP}")
                        elif npw != confirm_npw:
                            st.error("❌ Passwords do not match.")
                        else:
                            with get_db() as c:
                                c.execute("UPDATE users SET password_hash=? WHERE email=?",
                                          (hash_txt(npw), st.session_state.reset_email))
                            st.success("🎉 Password updated! Please sign in.")
                            time.sleep(1)
                            st.session_state.reset_mode = None
                            st.session_state.reset_email = None
                            st.session_state.fp_verified = False
                            navigate("Login")

            st.markdown("<br>", unsafe_allow_html=True)
            if st.button("← Cancel", use_container_width=True):
                st.session_state.reset_mode = None
                st.session_state.reset_email = None
                st.session_state.fp_verified = False
                navigate("Login")

else:
    payload = verify_jwt(st.session_state.token)
    if not payload:
        st.session_state.token = None
        st.session_state.page = "Login"
        st.rerun()

    email = payload["email"]
    is_admin = (email == ADMIN_EMAIL)

    if not is_admin:
        with get_db() as c:
            row = c.execute("SELECT username FROM users WHERE email=?", (email,)).fetchone()
        uname = row[0] if row else email
    else:
        uname = "Administrator"

    with st.sidebar:
        st.markdown(f"""
        <div style="padding:16px 8px;text-align:center;">
            <div style="font-size:28px;">⚡</div>
            <div style="font-weight:700;font-size:16px;color:{COLORS['text_heading']};">Freight Quote</div>
            <div style="font-size:11px;color:{COLORS['text_muted']};">{"Admin Panel" if is_admin else "Customer Portal"}</div>
        </div><hr style="border-color:{COLORS['border_light']};">
        """, unsafe_allow_html=True)

        opts = ["Dashboard", "Users", "Logout"] if is_admin else ["Dashboard", "Quotes", "Reports", "Logout"]
        icons = ["house", "people", "box-arrow-right"] if is_admin else ["house", "graph-up", "file-text", "box-arrow-right"]
        menu = option_menu(None, opts, icons=icons,
                            styles={"container": {"background-color": COLORS['bg_sidebar']},
                                    "nav-link-selected": {"background-color": COLORS['accent'], "color": "#ffffff"}})
        if menu == "Logout":
            st.session_state.token = None
            st.session_state.page = "Login"
            st.rerun()

    # ---------------- ADMIN DASHBOARD  ----------------
    if is_admin:
        st.markdown(f"""
        <div style="background:{COLORS['text_heading']};border-radius:16px;padding:24px 32px;display:flex;justify-content:space-between;align-items:center;margin-bottom:24px;">
            <div><h1 style="color:#ffffff !important;margin:0;font-size:24px !important;">⚡ Freight Quote</h1><div style="color:{COLORS['bg_card_alt']};font-size:13px;">Admin Control Panel</div></div>
            <div style="background:{COLORS['accent']};padding:8px 18px;border-radius:30px;font-weight:700;color:#ffffff;">🛡️ {uname}</div>
        </div>
        """, unsafe_allow_html=True)

        st.markdown("### 🛡️ Registered Users")
        with get_db() as c:
            users = c.execute("SELECT username, email FROM users ORDER BY id").fetchall()
        if users:
            st.table([{"Username": u, "Email": e} for u, e in users])  # passwords never displayed
        else:
            st.info("No users have registered yet.")

    # ---------------- REGULAR USER DASHBOARD ----------------
    else:
        st.markdown(f"""
        <div style="background:{COLORS['text_heading']};border-radius:16px;padding:24px 32px;display:flex;justify-content:space-between;align-items:center;margin-bottom:24px;">
            <div><h1 style="color:#ffffff !important;margin:0;font-size:24px !important;">⚡ Freight Quote</h1><div style="color:{COLORS['bg_card_alt']};font-size:13px;">Customer Dashboard</div></div>
            <div style="background:{COLORS['accent']};padding:8px 18px;border-radius:30px;font-weight:700;color:#ffffff;">👤 Welcome, {uname}</div>
        </div>
        """, unsafe_allow_html=True)

        c1, c2, c3, c4 = st.columns(4)
        for col, icon, lbl, val in [(c1, "📦", "Active Shipments", "12"), (c2, "⚡", "Quotes Today", "5"),
                                    (c3, "📊", "On-Time Rate", "97.8%"), (c4, "🛡️", "Account Status", "Secured")]:
            col.markdown(f"""
            <div class="pn-card" style="text-align:center;">
                <div style="font-size:28px;">{icon}</div>
                <div style="font-size:26px;font-weight:700;color:{COLORS['text_heading']};">{val}</div>
                <div style="color:{COLORS['text_muted']};font-size:12px;font-weight:600;">{lbl}</div>
            </div>
            """, unsafe_allow_html=True)

        st.markdown("<br>", unsafe_allow_html=True)
        fig = go.Figure(go.Indicator(mode="gauge+number", value=92,
                        title={"text": "Fleet Health Index", "font": {"color": COLORS['text_heading'], "size": 14}},
                        gauge={"axis": {"range": [0, 100]}, "bar": {"color": COLORS['accent']},
                               "bgcolor": COLORS['bg_card_alt'], "borderwidth": 1, "bordercolor": COLORS['border']}))
        fig.update_layout(paper_bgcolor="rgba(0,0,0,0)", font={"color": COLORS['text_main'], "family": "Inter"},
                           height=260, margin=dict(l=10, r=10, t=40, b=10))
        st.plotly_chart(fig, use_container_width=True)
