import os
import time
import json # Added for handling multiple image URLs as JSON
from flask import Flask, render_template_string, request, redirect, url_for, session, flash, Response
import mysql.connector
from werkzeug.security import generate_password_hash, check_password_hash
import cloudinary
import cloudinary.uploader
from flask_mail import Mail, Message
from dotenv import load_dotenv
import difflib # For fuzzy matching
from io import BytesIO
from reportlab.lib.pagesizes import letter
from reportlab.platypus import SimpleDocTemplate, Table, TableStyle, Image, Paragraph, Spacer
from reportlab.lib.styles import getSampleStyleSheet
from reportlab.lib import colors
from reportlab.lib.units import inch
load_dotenv()
app = Flask(__name__)
app.secret_key = os.getenv("FLASK_SECRET", "super_secret_key_123")
cloudinary.config(
    cloud_name=os.getenv("CLOUDINARY_NAME", "duifjklg7"),
    api_key=os.getenv("CLOUDINARY_KEY", "918878566914447"),
    api_secret=os.getenv("CLOUDINARY_SECRET", "u-dYpWFblTNEKCcl-gRfSZ_zdx0")
)
db_config = {
    "host": os.getenv("DB_HOST", "localhost"),
    "user": os.getenv("DB_USER", "flaskuser"),
    "password": os.getenv("DB_PASS", "flaskpass"),
    "database": os.getenv("DB_NAME", "lf_db")
}
app.config["MAIL_SERVER"] = "smtp.gmail.com"
app.config["MAIL_PORT"] = 587
app.config["MAIL_USE_TLS"] = True
app.config["MAIL_USERNAME"] = os.getenv("MAIL_USERNAME", "hkbka0302@gmail.com")
app.config["MAIL_PASSWORD"] = os.getenv("MAIL_PASSWORD", "sxcp cwgh mivz crtb")
app.config["MAIL_DEFAULT_SENDER"] = app.config["MAIL_USERNAME"]
mail = Mail(app)
def get_db():
    con = mysql.connector.connect(**db_config)
    con.autocommit = True # Enable autocommit to reduce lock hold times
    return con
def query_db(query, args=(), fetchone=False, commit=False, return_id=False, retries=3):
    last_error = None
    for attempt in range(retries):
        con = get_db()
        cur = con.cursor(dictionary=True)
        try:
            cur.execute(query, args)
            if commit:
                con.commit()
            if return_id:
                id_val = cur.lastrowid
                cur.close()
                con.close()
                return id_val
            result = cur.fetchone() if fetchone else cur.fetchall()
            cur.close()
            con.close()
            return result
        except mysql.connector.errors.DatabaseError as e:
            cur.close()
            con.close()
            if e.errno == 1205 and attempt < retries - 1: # Lock timeout
                wait_time = 0.1 * (2 ** attempt) # Exponential backoff: 0.1s, 0.2s, 0.4s
                time.sleep(wait_time)
                app.logger.warning(f"Lock wait timeout on attempt {attempt + 1}/{retries}. Retrying in {wait_time}s: {e}")
                last_error = e
                continue
            raise e # Re-raise if max retries or other error
    raise last_error # If all retries fail
def init_db():
    # Set session lock timeout higher to give more breathing room
    con = get_db()
    cur = con.cursor()
    cur.execute("SET SESSION innodb_lock_wait_timeout = 120") # 120 seconds
    con.commit()
    cur.close()
    con.close()
    query_db("""CREATE TABLE IF NOT EXISTS users(
            id INT AUTO_INCREMENT PRIMARY KEY,
            email VARCHAR(255) UNIQUE,
            first_name VARCHAR(100),
            last_name VARCHAR(100),
            password VARCHAR(255)
        )""", commit=True)
    # Updated to image_urls TEXT for multiple images (JSON array)
    query_db("""CREATE TABLE IF NOT EXISTS items(
            id INT AUTO_INCREMENT PRIMARY KEY,
            user_email VARCHAR(255),
            type VARCHAR(20),
            item_name VARCHAR(255),
            description TEXT,
            location VARCHAR(255),
            image_urls TEXT,
            phone VARCHAR(20),
            status VARCHAR(20) DEFAULT 'pending'
        )""", commit=True)
    query_db("""CREATE TABLE IF NOT EXISTS admin(
            id INT AUTO_INCREMENT PRIMARY KEY,
            username VARCHAR(100) UNIQUE,
            password VARCHAR(255)
        )""", commit=True)
    query_db("""CREATE TABLE IF NOT EXISTS notifications(
            id INT AUTO_INCREMENT PRIMARY KEY,
            user_email VARCHAR(255),
            message TEXT,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )""", commit=True)
    # Changed matches status to VARCHAR(20) to avoid ENUM truncation issues
    query_db("""CREATE TABLE IF NOT EXISTS matches(
            id INT AUTO_INCREMENT PRIMARY KEY,
            lost_item_id INT,
            found_item_id INT,
            lost_email VARCHAR(255),
            found_email VARCHAR(255),
            status VARCHAR(20) DEFAULT 'pending',
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (lost_item_id) REFERENCES items(id),
            FOREIGN KEY (found_item_id) REFERENCES items(id)
        )""", commit=True)
    # Migration: If table exists with ENUM, alter to VARCHAR to prevent truncation
    try:
        query_db("ALTER TABLE matches MODIFY COLUMN status VARCHAR(20) DEFAULT 'pending'", commit=True)
    except mysql.connector.errors.ProgrammingError:
        # Ignore if already VARCHAR or other non-critical error
        pass
    # Migration: Add image_urls if old table has image
    try:
        query_db("ALTER TABLE items ADD COLUMN image_urls TEXT AFTER image", commit=True)
        # Migrate existing image to image_urls as JSON array
        query_db("""
            UPDATE items
            SET image_urls = JSON_ARRAY(image)
            WHERE image IS NOT NULL AND image != ''
        """, commit=True)
        # Drop old image column after migration
        query_db("ALTER TABLE items DROP COLUMN image", commit=True)
    except mysql.connector.errors.ProgrammingError:
        # Ignore if already migrated or other error
        pass
    # Add phone column if not exists
    try:
        query_db("ALTER TABLE items ADD COLUMN phone VARCHAR(20)", commit=True)
    except mysql.connector.errors.ProgrammingError:
        pass
    con = get_db()
    cur = con.cursor(dictionary=True)
    cur.execute("SELECT * FROM admin WHERE username=%s", ("admin",))
    row = cur.fetchone()
    if not row:
        hashed = generate_password_hash(os.getenv("ADMIN_PASSWORD", "admin123")) # Use env for security
        cur2 = con.cursor()
        cur2.execute("INSERT INTO admin (username, password) VALUES (%s, %s)", ("admin", hashed))
        con.commit()
        cur2.close()
    cur.close()
    con.close()
init_db()
def fuzzy_match(str1, str2, threshold=0.6): # Increased threshold for better precision
    if not str1 or not str2 or len(str1) < 3 or len(str2) < 3:
        return False
    return difflib.SequenceMatcher(None, str1.lower(), str2.lower()).ratio() > threshold
def find_potential_matches(returned_found_items):
    lost_items = query_db("SELECT * FROM items WHERE type='Lost' AND status='pending'")
    matches = []
    for found in returned_found_items:
        f_text = f"{found.get('item_name', '')} {found.get('description', '')} {found.get('location', '')}".strip()
        if not f_text:
            continue
        for l in lost_items:
            l_text = f"{l.get('item_name', '')} {l.get('description', '')} {l.get('location', '')}".strip()
            if fuzzy_match(l_text, f_text):
                # Parse image_urls here
                if l.get('image_urls'):
                    l['image_urls_list'] = json.loads(l['image_urls'])
                else:
                    l['image_urls_list'] = []
                if found.get('image_urls'):
                    found['image_urls_list'] = json.loads(found['image_urls'])
                else:
                    found['image_urls_list'] = []
                matches.append({
                    "lost": l,
                    "found": found,
                    "score": difflib.SequenceMatcher(None, l_text.lower(), f_text.lower()).ratio()
                })
    return sorted(matches, key=lambda x: x['score'], reverse=True)
def send_email(to_email, subject, body):
    if not to_email or not app.config.get("MAIL_USERNAME") or not app.config.get("MAIL_PASSWORD"):
        app.logger.warning("Mail settings or recipient missing; skipping email send.")
        return False, "Mail not configured"
    try:
        msg = Message(subject, recipients=[to_email])
        msg.body = body
        mail.send(msg)
        return True, "Sent"
    except Exception as e:
        app.logger.exception("Failed to send email")
        return False, str(e)
[01/02, 7:42 pm] Pa 🫶: def get_safe_img_url(url):
    if url and isinstance(url, str) and '.avif' in url.lower():
        url = url.replace('/v', '/f_png/v').replace('.avif', '.png')
    return url
def get_base_style(page_key):
    if page_key == "auth":
        bg = "linear-gradient(135deg, #ff6ec7 0%, #7b2ff7 50%, #00c6ff 100%)"
    elif page_key == "home":
        bg = "linear-gradient(135deg, #ff6ec7 0%, #7b2ff7 50%, #00c6ff 100%)"
    elif page_key == "dashboard":
        bg = "linear-gradient(135deg, #ff6ec7 0%, #7b2ff7 45%, #00c6ff 100%)"
    elif page_key == "view":
        bg = "linear-gradient(135deg, #ff9a9e 0%, #fad0c4 50%, #7b2ff7 100%)"
    else:
        bg = "linear-gradient(135deg, #ff6ec7 0%, #7b2ff7 50%, #00c6ff 100%)"
    return f"""
<style>
:root {{ --bg-grad: {bg}; }}
html, body {{ height:100%; margin:0; }}
body {{
  font-family: 'Roboto', Arial, sans-serif;
  margin:0; padding:0;
  background: var(--bg-grad);
  background-attachment: fixed;
  color:#222;
  min-height:100vh;
  display:flex;
  align-items:flex-start;
  justify-content:center;
  padding:24px;
}}
.bg-overlay {{ background: rgba(255,255,255,0.06); backdrop-filter: blur(6px); position:fixed; inset:0; z-index:-1; }}
.container {{ width:100%; max-width:1100px; }}
.main {{
  max-width:1100px; margin:40px auto 0 auto; padding:24px;
  background:rgba(255,255,255,0.92);
  border-radius:18px; box-shadow:0 8px 40px rgba(0,0,0,0.12);
}}
.navbar {{ background: #fff; box-shadow: 0 2px 8px rgba(0,0,0,0.07); padding: 0 20px; height:64px; display:flex; align-items:center; justify-content:space-between; border-radius:12px; }}
.navbar .logo {{ font-size:1.4rem; font-weight:700; color:#6a11cb; letter-spacing:1px; }}
.navbar .user {{ font-size:1rem; color:#555; }}
.navbar .notif {{ font-size:2rem; color:#6a11cb; margin-right:18px; cursor:pointer; position:relative; text-decoration:none; display:flex; align-items:center; gap:8px; font-weight:600; }}
.navbar .notif .badge {{ position:absolute; top:-6px; right:-8px; background:#d32f2f; color:#fff; border-radius:50%; font-size:0.8rem; padding:2px 7px; min-width:18px; height:18px; display:flex; align-items:center; justify-content:center; }}
.card-grid {{ display:grid; grid-template-columns:repeat(auto-fit,minmax(320px,1fr)); gap:24px; margin-top:32px; align-items:start; }}
.card {{ background:#fff; border-radius:14px; box-shadow:0 4px 16px rgba(0,0,0,0.08); padding:20px; transition:0.3s all; position:relative; min-height:280px; overflow:hidden; display:flex; flex-direction:column; }}
.card:hover {{ transform:translateY(-4px); box-shadow:0 8px 32px rgba(106,17,203,0.15); }}
.card img {{ max-width:100%; max-height:120px; object-fit:cover; border-radius:8px; margin-top:auto; }} /* Fallback for single image */
.card .type {{ position:absolute; top:12px; right:12px; font-size:0.85rem; font-weight:600; padding:6px 12px; border-radius:20px; }}
.card.lost .type {{ background:#ffebee; color:#d32f2f; }} /* Red for lost/pending */
.card.found .type {{ background:#e8f5e9; color:#388e3c; }} /* Green for found */
.card.pending .type {{ background:#fff3e0; color:#f57c00; }} /* Orange for pending */
.card.returned .type {{ background:#e8f5e9; color:#388e3c; }} /* Green for returned/success */
.card.matched .type {{ background:#e8f5e9; color:#388e3c; }} /* Green for matched */
.card.collected .type {{ background:#e3f2fd; color:#1976d2; }} /* Blue for collected */
.btn {{ padding:10px 22px; border-radius:8px; font-weight:600; text-decoration:none; margin:8px 0; cursor:pointer; border:none; background:linear-gradient(90deg,#7b2ff7,#00c6ff); color:#fff; transition:0.25s; font-size:1.05rem; }}
.btn:hover {{ transform:translateY(-2px); opacity:0.95; }}
.btn.reject {{ background:linear-gradient(90deg,#ff8a65,#ff5252); }}
.btn.approve {{ background:linear-gradient(90deg,#a5d6a7,#388e3c); }}
.btn.return {{ background:linear-gradient(90deg,#e3f2fd,#bbdefb); color:#1976d2; }}
.btn.view {{ background:linear-gradient(90deg,#64b5f6,#1976d2); }}
/* Concise form inputs: tighter spacing */
input, textarea {{ width:100%; padding:8px; margin:5px 0; border-radius:6px; border:1px solid #e0e0e0; background:#fafafa; color:#222; box-sizing:border-box; font-size:0.95rem; }}
input:focus, textarea:focus {{ border-color:#7b2ff7; box-shadow:0 0 6px rgba(123,47,247,0.12); outline:none; }}
textarea {{ resize:vertical; min-height:60px; }} /* Compact textarea */
.form-compact {{ margin:0; padding:0; }} /* For tighter form */
.form-compact > * {{ margin:0 0 8px 0; }} /* Tighten child elements */
.footer {{ text-align:center; padding:18px; color:#555; font-size:0.95rem; margin-top:40px; }}
.edit-delete {{ position:absolute; top:12px; left:12px; display:flex; gap:8px; }}
.edit-delete a {{ font-size:1.1rem; padding:4px 8px; border-radius:6px; text-decoration:none; transition:0.2s; background:#f3f3f3; color:#6a11cb; }}
.edit-delete a:hover {{ background:#e3f2fd; }}
.card-stats {{ display:grid; grid-template-columns:repeat(auto-fit,minmax(200px,1fr)); gap:16px; margin-bottom:32px; }}
.stat-card {{ background:linear-gradient(90deg,#f3e8ff,#e0f7ff); color:#6a11cb; text-align:center; padding:20px; border-radius:12px; font-weight:700; box-shadow:0 4px 12px rgba(106,17,203,0.08); transition:0.3s; cursor:pointer; }}
.stat-card:hover {{ transform:translateY(-4px); box-shadow:0 8px 24px rgba(106,17,203,0.15); }}
.stat-card.lost {{ background:linear-gradient(90deg,#ffebee,#ffcdd2); color:#d32f2f; }}
.stat-card.found {{ background:linear-gradient(90deg,#e8f5e9,#c8e6c9); color:#388e3c; }}
.stat-card.pending {{ background:linear-gradient(90deg,#fff3e0,#ffe0b2); color:#f57c00; }}
.stat-card.match {{ background:linear-gradient(90deg,#fffde7,#fff9c4); color:#fbc02d; }}
.stat-card .icon {{ font-size:2.5rem; display:block; margin-bottom:12px; }}
.stat-card h3 {{ font-size:1.2rem; margin:0 0 8px 0; opacity:0.8; }}
.stat-card .count {{ font-size:2.8rem; font-weight:900; line-height:1; }}
.quick_actions {{ display:flex; flex-wrap:wrap; gap:16px; margin-bottom:32px; justify-content:center; }}
.dashboard_header {{ display:flex; align-items:center; justify-content:space-between; margin-bottom:24px; }}
.dashboard_header h2 {{ color:#6a11cb; font-size:2.2rem; font-weight:800; margin:0; }}
.dashboard_header .college {{ font-size:1.1rem; color:#388e3c; font-weight:600; }}
.match_card {{ background:#fff; border-radius:14px; box-shadow:0 2px 12px rgba(0,0,0,0.07); padding:22px; margin-bottom:20px; }}
.match_actions {{ display:flex; gap:12px; margin-top:12px; }}
.pending_matches {{ background:rgba(255,255,255,0.92); padding:20px; border-radius:12px; margin:20px 0; }}
{{-- New styles for multiple small images in a row and improved match display --}}
.images-row {{
  display: flex;
  gap: 10px; /* Space between images */
  overflow-x: auto; /* Horizontal scroll if many images */
  padding: 10px 0;
  border-top: 1px solid #eee; /* Subtle divider */
  margin-top: auto;
}}
.thumbnail {{
  width: 80px;
  height: 80px;
  object-fit: cover; /* Keeps aspect ratio, crops if needed */
  border-radius: 4px;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1); /* Light shadow */
  flex-shrink: 0; /* Prevents squishing */
}}
.match-images {{
  display: flex;
  justify-content: space-between;
  gap: 10px;
  margin: 10px 0;
}}
.match-images .images-row {{
  flex: 1;
  border-top: none;
  padding: 0;
}}
.match-label {{
  font-weight: 500;
  font-size: 0.9em;
  color: #666;
  margin-bottom: 5px;
}}
.admin-notifs {{ background:#fff3e0; padding:12px; border-radius:8px; margin:16px 0; }}
.admin-notif {{ padding:8px; border-bottom:1px solid #eee; font-size:0.9rem; }}
.admin-notif:last-child {{ border-bottom:none; }}
/* Match column enhancements: Compact match preview in cards */
.match-preview {{
  background: #f0f8ff;
  border-left: 3px solid #388e3c;
  padding: 8px;
  margin-top: 8px;
  border-radius: 4px;
  font-size: 0.85rem;
}}
.match-preview h5 {{ margin: 0 0 4px 0; font-size: 0.9rem; color: #388e3c; }}
.match-preview small {{ color: #666; }}
@media (max-width:700px) {{
  .main {{ padding:12px; border-radius:12px; margin-top:20px; }}
  .card-grid {{ grid-template-columns:1fr; gap:16px; }}
  .navbar {{ padding:0 12px; height:56px; }}
  .match_actions {{ flex-direction:column; }}
  .quick_actions {{ flex-direction:column; align-items:center; }}
  .card {{ padding:16px; min-height:260px; }}
  .dashboard_header {{ flex-direction:column; gap:8px; text-align:center; }}
  .thumbnail {{ width: 60px; height: 60px; }}
  .match-images {{ flex-direction: column; }}
}}
.report-stats {{ display: grid; grid-template-columns: repeat(auto-fit, minmax(250px, 1fr)); gap: 20px; margin: 20px 0; }}
.report-stat {{ background: #f9f9f9; padding: 20px; border-radius: 10px; text-align: center; border-left: 4px solid #6a11cb; }}
.report-stat h3 {{ margin: 0 0 10px 0; color: #6a11cb; }}
.report-stat .count {{ font-size: 2rem; font-weight: bold; color: #388e3c; }}
.download-btn {{ display: block; margin: 20px auto; padding: 12px 24px; background: linear-gradient(90deg, #388e3c, #4caf50); color: white; text-decoration: none; border-radius: 8px; font-weight: bold; }}
</style>
<div class="bg-overlay"></div>
"""
# Admin routes (concise dashboard)
@app.route("/admin_login", methods=["GET", "POST"])
def admin_login():
    if session.get("admin"):
        return redirect(url_for("admin_dashboard"))
    if request.method == "POST":
        username = request.form.get("username")
        password = request.form.get("password")
        if not username or not password:
            flash("Username and password required.")
            return render_template_string(get_base_style("auth") + """...""") # Abbrev for brevity
        admin = query_db("SELECT * FROM admin WHERE username=%s", (username,), fetchone=True)
        if admin:
            stored = admin.get("password") or ""
            try:
                if check_password_hash(stored, password):
                    session.clear()
                    session["admin"] = True
                    session["admin_username"] = admin["username"]
                    return redirect(url_for("admin_dashboard"))
            except ValueError:
                pass
            if stored == password:
                new_hashed = generate_password_hash(password)
                con = get_db()
                cur = con.cursor()
                cur.execute("UPDATE admin SET password=%s WHERE id=%s", (new_hashed, admin["id"]))
                con.commit()
                cur.close()
                con.close()
                session.clear()
                session["admin"] = True
                session["admin_username"] = admin["username"]
                return redirect(url_for("admin_dashboard"))
        flash("Invalid admin credentials.")
    return render_template_string(get_base_style("auth") + """
<div class="container">
<div class='navbar'>
  <span class='logo'>Lost & Found Portal - Admin</span>
</div>
<div class='main' style='max-width:440px;margin-top:80px;'>
  <h2>Admin Login</h2>
  <form method="POST" autocomplete="off">
    <input name="username" placeholder="Username" required autocomplete="new-username"><br>
    <input name="password" type="password" placeholder="Password" required autocomplete="new-password"><br>
    <button class="btn" type="submit">Login</button>
[01/02, 7:42 pm] Pa 🫶: </form>
  <p style="margin-top:12px;"><a href="{{ url_for('welcome') }}">← Back to site</a></p>
  {% with messages = get_flashed_messages() %}{% if messages %}<ul>{% for m in messages %}<li>{{ m }}</li>{% endfor %}</ul>{% endif %}{% endwith %}
</div>
</div>
""")
@app.route("/admin_dashboard")
def admin_dashboard():
    if not session.get("admin"):
        return redirect(url_for("admin_login"))
    pending_returns = query_db("SELECT COUNT(*) as cnt FROM items WHERE type='Found' AND status='pending'", fetchone=True)["cnt"]
    pending_matches_count = query_db("SELECT COUNT(*) as cnt FROM matches WHERE status='pending'", fetchone=True)["cnt"]
    total_lost = query_db("SELECT COUNT(*) as cnt FROM items WHERE type='Lost'", fetchone=True)["cnt"]
    total_found = query_db("SELECT COUNT(*) as cnt FROM items WHERE type='Found'", fetchone=True)["cnt"]
    return render_template_string(get_base_style("dashboard") + """
<div class="container">
<div class='navbar'>
  <span class='logo'>Lost & Found Portal - Admin</span>
  <span style="display:flex;align-items:center;">
    <span class='user'>Admin: {{ session.get('admin_username') }} | <a href="{{ url_for('admin_logout') }}" class="btn" style="background:#444;">Logout</a></span>
  </span>
</div>
<div class='main'>
  <div class="dashboard-header">
    <h2>Admin Dashboard</h2>
    <span class="college">AIML Department</span>
  </div>
  <div class="card-stats">
    <a href="{{ url_for('admin_returns') }}" class="stat-card pending">
      <span class="icon">📦</span>
      <h3>Pending Returns</h3>
      <div class="count">{{ pending_returns }}</div>
    </a>
    <a href="{{ url_for('admin_matches') }}" class="stat-card match">
      <span class="icon">🤝</span>
      <h3>Pending Matches</h3>
      <div class="count">{{ pending_matches_count }}</div>
    </a>
    <a href="{{ url_for('admin_full_view', tab='lost') }}" class="stat-card lost">
      <span class="icon">🛑</span>
      <h3>Total Lost</h3>
      <div class="count">{{ total_lost }}</div>
    </a>
    <a href="{{ url_for('admin_full_view', tab='found') }}" class="stat-card found">
      <span class="icon">🔍</span>
      <h3>Total Found</h3>
      <div class="count">{{ total_found }}</div>
    </a>
  </div>
  <div class="quick-actions">
    <a href="{{ url_for('admin_returns') }}" class="btn approve">Manage Returns</a>
    <a href="{{ url_for('admin_matches') }}" class="btn match">Manage Matches</a>
    <a href="{{ url_for('admin_full_view') }}" class="btn view">View All Items</a>
    <a href="{{ url_for('admin_report') }}" class="btn" style="background:linear-gradient(90deg,#ff9800,#ffc107);">Generate Report</a>
  </div>
</div>
<div class='footer'>© 2025 Lost & Found Portal (Admin)</div>
</div>
""", pending_returns=pending_returns, pending_matches_count=pending_matches_count, total_lost=total_lost, total_found=total_found)
@app.route("/admin_report")
def admin_report():
    if not session.get("admin"):
        return redirect(url_for("admin_login"))
    # Fetch stats
    pending_lost = query_db("SELECT COUNT(*) as cnt FROM items WHERE type='Lost' AND status='pending'", fetchone=True)["cnt"]
    pending_found = query_db("SELECT COUNT(*) as cnt FROM items WHERE type='Found' AND status='pending'", fetchone=True)["cnt"]
    matched_count = query_db("SELECT COUNT(*) as cnt FROM matches WHERE status IN ('approved', 'collected')", fetchone=True)["cnt"]
    total_lost = query_db("SELECT COUNT(*) as cnt FROM items WHERE type='Lost'", fetchone=True)["cnt"]
    total_found = query_db("SELECT COUNT(*) as cnt FROM items WHERE type='Found'", fetchone=True)["cnt"]
    returned_found = query_db("SELECT COUNT(*) as cnt FROM items WHERE type='Found' AND status IN ('returned', 'matched', 'collected')", fetchone=True)["cnt"]
    # Fetch lists for display (limited for page)
    pending_lost_items = query_db("SELECT id, item_name, description, location, user_email FROM items WHERE type='Lost' AND status='pending' ORDER BY id DESC LIMIT 10")
    pending_found_items = query_db("SELECT id, item_name, description, location, user_email FROM items WHERE type='Found' AND status='pending' ORDER BY id DESC LIMIT 10")
    return render_template_string(get_base_style("dashboard") + """
<div class="container">
<div class='navbar'>
  <span class='logo'>Lost & Found Portal - Admin</span>
  <span style="display:flex;align-items:center;">
    <a href="{{ url_for('admin_dashboard') }}" class="btn">← Back to Dashboard</a>
  </span>
</div>
<div class='main'>
  <h2>Report Overview</h2>
  <p style="margin-bottom:20px;">Current statistics (updates dynamically as items are processed).</p>
  <div class="report-stats">
    <div class="report-stat">
      <h3>Pending Lost (To Be Found)</h3>
      <div class="count">{{ pending_lost }}</div>
    </div>
    <div class="report-stat">
      <h3>Pending Found (To Be Returned)</h3>
      <div class="count">{{ pending_found }}</div>
    </div>
    <div class="report-stat">
      <h3>Matched Items</h3>
      <div class="count">{{ matched_count }}</div>
    </div>
    <div class="report-stat">
      <h3>Total Lost</h3>
      <div class="count">{{ total_lost }}</div>
    </div>
    <div class="report-stat">
      <h3>Total Found</h3>
      <div class="count">{{ total_found }}</div>
    </div>
    <div class="report-stat">
      <h3>Handled Found</h3>
      <div class="count">{{ returned_found }}</div>
    </div>
  </div>
  <a href="{{ url_for('admin_report_pdf') }}" class="download-btn">📄 Download PDF Report</a>
  <h3 style="margin-top:40px;">Recent Pending Lost Items</h3>
  <div style="overflow:auto; margin-top:10px;">
    <table style="width:100%;border-collapse:collapse;">
      <tr style="background:#ffebee;">
        <th style="padding:10px;border:1px solid #e0e0e0;">ID</th>
        <th style="padding:10px;border:1px solid #e0e0e0;">Name</th>
        <th style="padding:10px;border:1px solid #e0e0e0;">Description</th>
        <th style="padding:10px;border:1px solid #e0e0e0;">Location</th>
        <th style="padding:10px;border:1px solid #e0e0e0;">User</th>
      </tr>
      {% for i in pending_lost_items %}
      <tr>
        <td style="padding:8px;border:1px solid #eee;">{{ i['id'] }}</td>
        <td style="padding:8px;border:1px solid #eee;">{{ i['item_name'] }}</td>
        <td style="padding:8px;border:1px solid #eee;">{{ i['description'][:50] }}...</td>
        <td style="padding:8px;border:1px solid #eee;">{{ i['location'] }}</td>
        <td style="padding:8px;border:1px solid #eee;">{{ i['user_email'] }}</td>
      </tr>
      {% endfor %}
    </table>
  </div>
  <h3 style="margin-top:40px;">Recent Pending Found Items</h3>
  <div style="overflow:auto; margin-top:10px;">
    <table style="width:100%;border-collapse:collapse;">
      <tr style="background:#e8f5e9;">
        <th style="padding:10px;border:1px solid #e0e0e0;">ID</th>
        <th style="padding:10px;border:1px solid #e0e0e0;">Name</th>
        <th style="padding:10px;border:1px solid #e0e0e0;">Description</th>
        <th style="padding:10px;border:1px solid #e0e0e0;">Location</th>
        <th style="padding:10px;border:1px solid #e0e0e0;">User</th>
      </tr>
      {% for i in pending_found_items %}
      <tr>
        <td style="padding:8px;border:1px solid #eee;">{{ i['id'] }}</td>
        <td style="padding:8px;border:1px solid #eee;">{{ i['item_name'] }}</td>
        <td style="padding:8px;border:1px solid #eee;">{{ i['description'][:50] }}...</td>
        <td style="padding:8px;border:1px solid #eee;">{{ i['location'] }}</td>
        <td style="padding:8px;border:1px solid #eee;">{{ i['user_email'] }}</td>
      </tr>
      {% endfor %}
    </table>
  </div>
</div>
<div class='footer'>© 2025 Lost & Found Portal (Admin)</div>
</div>
""", pending_lost=pending_lost, pending_found=pending_found, matched_count=matched_count, total_lost=total_lost, total_found=total_found, returned_found=returned_found, pending_lost_items=pending_lost_items, pending_found_items=pending_found_items)
@app.route("/admin_report_pdf")
def admin_report_pdf():
    if not session.get("admin"):
        return redirect(url_for("admin_login"))
    # Fetch stats
    pending_lost = query_db("SELECT COUNT(*) as cnt FROM items WHERE type='Lost' AND status='pending'", fetchone=True)["cnt"]
    pending_found = query_db("SELECT COUNT(*) as cnt FROM items WHERE type='Found' AND status='pending'", fetchone=True)["cnt"]
    matched_count = query_db("SELECT COUNT(*) as cnt FROM matches WHERE status IN ('approved', 'collected')", fetchone=True)["cnt"]
    total_lost = query_db("SELECT COUNT(*) as cnt FROM items WHERE type='Lost'", fetchone=True)["cnt"]
    total_found = query_db("SELECT COUNT(*) as cnt FROM items WHERE type='Found'", fetchone=True)["cnt"]
    returned_found = query_db("SELECT COUNT(*) as cnt FROM items WHERE type='Found' AND status IN ('returned', 'matched', 'collected')", fetchone=True)["cnt"]
    # Fetch full lists for PDF (limit to 10 per for conciseness)
    pending_lost_items = query_db("SELECT id, item_name, description, location, user_email, image_urls FROM items WHERE type='Lost' AND status='pending' ORDER BY id DESC LIMIT 10")
    pending_found_items = query_db("SELECT id, item_name, description, location, user_email, image_urls FROM items WHERE type='Found' AND status='pending' ORDER BY id DESC LIMIT 10")
    # Updated query for handled matches (paired lost and found)
    handled_matches = query_db("""
        SELECT m.id as match_id, li.id as lost_id, li.item_name as lost_name, li.description as lost_desc, li.location as lost_loc, li.user_email as lost_email, li.phone as lost_phone, li.image_urls as lost_images,
               fi.id as found_id, fi.item_name as found_name, fi.description as found_desc, fi.location as found_loc, fi.user_email as found_email, fi.phone as found_phone, fi.image_urls as found_images,
               m.status, m.created_at
        FROM matches m
        JOIN items li ON m.lost_item_id = li.id
        JOIN items fi ON m.found_item_id = fi.id
        WHERE m.status IN ('approved', 'collected')
        ORDER BY m.created_at DESC LIMIT 10
    """)
    buffer = BytesIO()
    doc = SimpleDocTemplate(buffer, pagesize=letter)
    story = []
    styles = getSampleStyleSheet()
    # Title
    title = Paragraph("Lost & Found Report - AIML Department", styles['Title'])
    story.append(title)
    story.append(Spacer(1, 12))
    story.append(Paragraph(f"Generated: {time.strftime('%Y-%m-%d %H:%M:%S')}", styles['Normal']))
    story.append(Spacer(1, 24))
    # Stats Table (concise)
    stats_data = [
        ['Metric', 'Count'],
        ['Pending Lost', pending_lost],
        ['Pending Found', pending_found],
        ['Matched Items', matched_count],
        ['Total Lost', total_lost],
        ['Total Found', total_found],
[01/02, 7:42 pm] Pa 🫶: ['Handled Found', returned_found]
    ]
    stats_table = Table(stats_data, colWidths=[3*inch, 1*inch])
    stats_table.setStyle(TableStyle([
        ('BACKGROUND', (0, 0), (-1, 0), colors.grey),
        ('TEXTCOLOR', (0, 0), (-1, 0), colors.whitesmoke),
        ('ALIGN', (0, 0), (-1, -1), 'LEFT'),
        ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
        ('FONTSIZE', (0, 0), (-1, 0), 12),
        ('BOTTOMPADDING', (0, 0), (-1, 0), 12),
        ('BACKGROUND', (0, 1), (-1, -1), colors.beige),
        ('GRID', (0, 0), (-1, -1), 1, colors.black)
    ]))
    story.append(stats_table)
    story.append(Spacer(1, 24))
    # Pending Lost Table with proper Image (updated for alignment: larger columns, full desc wrap, bigger images)
    lost_header = Paragraph("Pending Lost Items (Top 10)", styles['Heading2'])
    story.append(lost_header)
    lost_data = [['ID', 'Image', 'Name', 'Description', 'Location', 'User']]
    for item in pending_lost_items:
        # Full description with wrapping (no snip)
        desc_para = Paragraph(item['description'], styles['Normal'])
        img_url = None
        if item['image_urls']:
            try:
                urls = json.loads(item['image_urls'])
                if urls:
                    img_url = urls[0]
            except json.JSONDecodeError:
                pass
        img_url = get_safe_img_url(img_url)
        img_cell = None
        if img_url:
            try:
                img = Image(img_url, width=0.8*inch, height=0.8*inch) # Bigger image
                img.hAlign = 'CENTER'
                img_cell = img
            except:
                img_cell = Paragraph('No Image', styles['Normal'])
        else:
            img_cell = Paragraph('No Image', styles['Normal'])
        lost_data.append([str(item['id']), img_cell, item['item_name'], desc_para, item['location'], item['user_email']])
    # Larger colWidths for better alignment and space
    lost_table = Table(lost_data, colWidths=[0.5*inch, 0.8*inch, 1.2*inch, 2.5*inch, 1.5*inch, 1.5*inch])
    lost_table.setStyle(TableStyle([
        ('BACKGROUND', (0, 0), (-1, 0), colors.lightgrey),
        ('ALIGN', (0, 0), (-1, -1), 'LEFT'),
        ('VALIGN', (0, 0), (-1, -1), 'TOP'), # Top align for wrapping text
        ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
        ('FONTSIZE', (0, 0), (-1, 0), 10),
        ('FONTSIZE', (0, 1), (-1, -1), 9), # Slightly larger font
        ('GRID', (0, 0), (-1, -1), 0.5, colors.black),
        ('LEFTPADDING', (0, 0), (-1, -1), 6), # More padding
        ('RIGHTPADDING', (0, 0), (-1, -1), 6),
        ('TOPPADDING', (0, 0), (-1, -1), 4),
        ('BOTTOMPADDING', (0, 0), (-1, -1), 4)
    ]))
    story.append(lost_table)
    story.append(Spacer(1, 24))
    # Pending Found Table with proper Image (similar, updated)
    found_header = Paragraph("Pending Found Items (Top 10)", styles['Heading2'])
    story.append(found_header)
    found_data = [['ID', 'Image', 'Name', 'Description', 'Location', 'User']]
    for item in pending_found_items:
        # Full description with wrapping
        desc_para = Paragraph(item['description'], styles['Normal'])
        img_url = None
        if item['image_urls']:
            try:
                urls = json.loads(item['image_urls'])
                if urls:
                    img_url = urls[0]
            except json.JSONDecodeError:
                pass
        img_url = get_safe_img_url(img_url)
        img_cell = None
        if img_url:
            try:
                img = Image(img_url, width=0.8*inch, height=0.8*inch) # Bigger
                img.hAlign = 'CENTER'
                img_cell = img
            except:
                img_cell = Paragraph('No Image', styles['Normal'])
        else:
            img_cell = Paragraph('No Image', styles['Normal'])
        found_data.append([str(item['id']), img_cell, item['item_name'], desc_para, item['location'], item['user_email']])
    # Larger colWidths
    found_table = Table(found_data, colWidths=[0.5*inch, 0.8*inch, 1.2*inch, 2.5*inch, 1.5*inch, 1.5*inch])
    found_table.setStyle(TableStyle([
        ('BACKGROUND', (0, 0), (-1, 0), colors.lightgrey),
        ('ALIGN', (0, 0), (-1, -1), 'LEFT'),
        ('VALIGN', (0, 0), (-1, -1), 'TOP'), # Top align
        ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
        ('FONTSIZE', (0, 0), (-1, 0), 10),
        ('FONTSIZE', (0, 1), (-1, -1), 9),
        ('GRID', (0, 0), (-1, -1), 0.5, colors.black),
        ('LEFTPADDING', (0, 0), (-1, -1), 6),
        ('RIGHTPADDING', (0, 0), (-1, -1), 6),
        ('TOPPADDING', (0, 0), (-1, -1), 4),
        ('BOTTOMPADDING', (0, 0), (-1, -1), 4)
    ]))
    story.append(found_table)
    story.append(Spacer(1, 24))
    # Handled Matches Table with proper Images and paired info (updated for compactness: smaller images, truncated desc, reduced widths/padding, combined contact, smaller fonts)
    handled_header = Paragraph("Handled Matches (Top 10)", styles['Heading2'])
    story.append(handled_header)
    handled_data = [['ID', 'Lost Img', 'Lost Item', 'Lost Contact', 'Found Img', 'Found Item', 'Found Contact', 'Status']]
    for match in handled_matches:
        # Truncated descriptions (50 chars max) for compactness
        lost_desc_short = match['lost_desc'][:50] + '...' if len(match['lost_desc']) > 50 else match['lost_desc']
        found_desc_short = match['found_desc'][:50] + '...' if len(match['found_desc']) > 50 else match['found_desc']
        lost_item_para = Paragraph(f"<b>{match['lost_name']}</b><br/><i>{lost_desc_short}</i>", styles['Normal'])
        found_item_para = Paragraph(f"<b>{match['found_name']}</b><br/><i>{found_desc_short}</i>", styles['Normal'])
        # Compact contact: email / phone in one line
        lost_contact_para = Paragraph(f"{match['lost_email']} / {match.get('lost_phone', 'N/A')}", styles['Normal'])
        found_contact_para = Paragraph(f"{match['found_email']} / {match.get('found_phone', 'N/A')}", styles['Normal'])
        # Smaller images (0.5 inch)
        lost_img_url = None
        if match['lost_images']:
            try:
                urls = json.loads(match['lost_images'])
                if urls:
                    lost_img_url = urls[0]
            except json.JSONDecodeError:
                pass
        lost_img_url = get_safe_img_url(lost_img_url)
        lost_img_cell = None
        if lost_img_url:
            try:
                img = Image(lost_img_url, width=0.5*inch, height=0.5*inch)
                img.hAlign = 'CENTER'
                lost_img_cell = img
            except:
                lost_img_cell = Paragraph('Img', styles['Normal'])
        else:
            lost_img_cell = Paragraph('Img', styles['Normal'])
        # Found image (smaller)
        found_img_url = None
        if match['found_images']:
            try:
                urls = json.loads(match['found_images'])
                if urls:
                    found_img_url = urls[0]
            except json.JSONDecodeError:
                pass
        found_img_url = get_safe_img_url(found_img_url)
        found_img_cell = None
        if found_img_url:
            try:
                img = Image(found_img_url, width=0.5*inch, height=0.5*inch)
                img.hAlign = 'CENTER'
                found_img_cell = img
            except:
                found_img_cell = Paragraph('Img', styles['Normal'])
        else:
            found_img_cell = Paragraph('Img', styles['Normal'])
        handled_data.append([
            str(match['match_id']),
            lost_img_cell,
            lost_item_para,
            lost_contact_para,
            found_img_cell,
            found_item_para,
            found_contact_para,
            match['status']
        ])
    # Reduced colWidths for compactness and better fit (total ~6.5 inch)
    handled_table = Table(handled_data, colWidths=[0.4*inch, 0.5*inch, 1.2*inch, 1.0*inch, 0.5*inch, 1.2*inch, 1.0*inch, 0.7*inch])
    handled_table.setStyle(TableStyle([
        ('BACKGROUND', (0, 0), (-1, 0), colors.green),
        ('ALIGN', (0, 0), (-1, -1), 'LEFT'),
        ('VALIGN', (0, 0), (-1, -1), 'MIDDLE'), # Middle align for compact rows
        ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
        ('FONTSIZE', (0, 0), (-1, 0), 9), # Smaller header font
        ('FONTSIZE', (0, 1), (-1, -1), 8), # Smaller body font
        ('GRID', (0, 0), (-1, -1), 0.5, colors.black),
        ('BACKGROUND', (0, 1), (-1, -1), colors.lightgreen),
        ('LEFTPADDING', (0, 0), (-1, -1), 3), # Reduced padding
        ('RIGHTPADDING', (0, 0), (-1, -1), 3),
        ('TOPPADDING', (0, 0), (-1, -1), 2), # Tighter vertical padding
        ('BOTTOMPADDING', (0, 0), (-1, -1), 2)
    ]))
    story.append(handled_table)
    doc.build(story)
    buffer.seek(0)
    response = Response(buffer.getvalue(), mimetype='application/pdf')
    response.headers['Content-Disposition'] = 'attachment; filename=lf_report.pdf'
    return response
@app.route("/admin_full_view")
def admin_full_view():
    if not session.get("admin"):
        return redirect(url_for("admin_login"))
    tab = request.args.get("tab", "all")
    status_filter = request.args.get("status", "all")
    valid_statuses = ['pending', 'returned', 'matched', 'collected', 'all']
    if status_filter != "all" and status_filter not in valid_statuses:
        status_filter = "all"
    if tab == "lost" and status_filter != "all":
        items = query_db("SELECT * FROM items WHERE type='Lost' AND status=%s ORDER BY id DESC", (status_filter,))
    elif tab == "found" and status_filter != "all":
        items = query_db("SELECT * FROM items WHERE type='Found' AND status=%s ORDER BY id DESC", (status_filter,))
    elif status_filter != "all":
        items = query_db("SELECT * FROM items WHERE status=%s ORDER BY id DESC", (status_filter,))
    elif tab == "lost":
        items = query_db("SELECT * FROM items WHERE type='Lost' ORDER BY id DESC")
    elif tab == "found":
        items = query_db("SELECT * FROM items WHERE type='Found' ORDER BY id DESC")
    else:
        items = query_db("SELECT * FROM items ORDER BY id DESC")
    return render_template_string(get_base_style("dashboard") + """
<div class="container">
<div class='navbar'>
  <span class='logo'>Lost & Found Portal - Admin</span>
  <span style="display:flex;align-items:center;">
    <a href="{{ url_for('admin_dashboard') }}" class="btn">← Back to Dashboard</a>
  </span>
</div>
<div class='main'>
  <h2>Full Items View</h2>
  <div style="overflow:auto; margin-top:20px;">
    <table style="width:100%;border-collapse:collapse;">
      <tr style="background:#6a11cb;color:#fff;">
        <th style="padding:10px;border:1px solid #e0e0e0;">ID</th>
        <th style="padding:10px;border:1px solid #e0e0e0;">Type</th>
        <th style="padding:10px;border:1px solid #e0e0e0;">Status</th>
        <th style="padding:10px;border:1px solid #e0e0e0;">Name</th>
        <th style="padding:10px;border:1px solid #e0e0e0;">User</th>
        <th style="padding:10px;border:1px solid #e0e0e0;">Phone</th>
        <th style="padding:10px;border:1px solid #e0e0e0;">Action</th>
      </tr>
      {% for i in items %}
      <tr>
        <td style="padding:8px;border:1px solid #eee;">{{ i['id'] }}</td>
        <td style="padding:8px;border:1px solid #eee;">{{ i['type'] }}</td>
        <td style="padding:8px;border:1px solid #eee;">{{ i['status'] }}</td>
        <td style="padding:8px;border:1px solid #eee;">{{ i['item_name'] }}</td>
        <td style="padding:8px;border:1px solid #eee;">{{ i['user_email'] }}</td>
        <td style="padding:8px;border:1px solid #eee;">{{ i['phone'] or 'N/A' }}</td>
        <td style="padding:8px;border:1px solid #eee;">
          <a href="{{ url_for('admin_delete', item_id=i['id']) }}" onclick="return confirm('Delete? This will also remove related matches.')" class="btn" style="background:#d32f2f;color:#fff;padding:6px 10px;font-size:0.9rem;">Delete</a>
        </td>
      </tr>
      {% endfor %}
    </table>
  </div>
</div>
<div class='footer'>© 2025 Lost & Found Portal (Admin)</div>
</div>
""", items=items)
@app.route("/admin_returns")
def admin_returns():
    if not session.get("admin"):
        return redirect(url_for("admin_login"))
    pending_returns = query_db("SELECT * FROM items WHERE type='Found' AND status='pending' ORDER BY id DESC")
    # Parse image_urls to list
    for item in pending_returns:
        if item.get('image_urls'):
            item['image_urls_list'] = json.loads(item['image_urls'])
        else:
            item['image_urls_list'] = []
    return render_template_string(get_base_style("dashboard") + """
<div class="container">
<div class='navbar'>
  <span class='logo'>Lost & Found Portal - Admin</span>
  <span style="display:flex;align-items:center;">
    <a href="{{ url_for('admin_dashboard') }}" class="btn">← Back to Dashboard</a>
  </span>
</div>
<div class='main'>
  <h2>Pending Returns</h2>
  <div class="card-grid">
    {% for item in pending_returns %}
    <div class="card found pending">
      <span class="type">Pending Return</span>
      <strong>{{ item['item_name'] }}</strong>
      <p style="margin:8px 0; line-height:1.4;">{{ item['description'] }}</p>
      <small style="color:#666;">Location: {{ item['location'] }}</small>
      {% if item['phone'] %}
      <small style="color:#666; display:block; margin-top:4px;">Phone: <b>{{ item['phone'] }}</b></small>
      {% endif %}
      {% if item['image_urls_list'] %}
        <div class="images-row">
          {% for url in item['image_urls_list'] %}
          <img src="{{ url }}" alt="Item Image" class="thumbnail">
          {% endfor %}
        </div>
      {% endif %}
      <div class="match-actions" style="margin-top:auto; padding-top:10px;">
        <a href="{{ url_for('admin_approve_return', item_id=item['id']) }}" class="btn approve">Approve Return & Notify Finder</a>
      </div>
    </div>
    {% endfor %}
  </div>
  {% if not pending_returns %}
  <p style="text-align:center; color:#666; padding:40px; font-style:italic;">No pending returns at the moment.</p>
  {% endif %}
</div>
<div class='footer'>© 2025 Lost & Found Portal (Admin)</div>
</div>
""", pending_returns=pending_returns)
[01/02, 7:42 pm] Pa 🫶: @app.route("/admin_approve_return/<int:item_id>")
def admin_approve_return(item_id):
    if not session.get("admin"):
        return redirect(url_for("admin_login"))
    item = query_db("SELECT * FROM items WHERE id=%s", (item_id,), fetchone=True)
    if item and item['type'] == 'Found' and item['status'] == 'pending':
        query_db("UPDATE items SET status='returned' WHERE id=%s", (item_id,), commit=True)
        finder_email = item['user_email']
        subject = "Item Return Approved"
        body = f"""Hi,
Your found item '{item['item_name']}' is approved. Return to AIML admin office.
Desc: {item['description'][:100]}
Loc: {item['location']}
Phone: {item.get('phone', 'N/A')}
AIML Lost & Found."""
        ok, info = send_email(finder_email, subject, body)
        note = f"Return approved and email to finder: {info}" if ok else f"Return approved but email failed: {info}"
        query_db("INSERT INTO notifications (user_email, message) VALUES (%s, %s)", (finder_email, note), commit=True)
    else:
        flash("Item not found or not pending.")
    return redirect(url_for("admin_returns"))
@app.route("/admin_matches")
def admin_matches():
    if not session.get("admin"):
        return redirect(url_for("admin_login"))
    returned_found = query_db("SELECT * FROM items WHERE type='Found' AND status='returned'")
    potential_matches = find_potential_matches(returned_found)
    pending_matches = query_db("SELECT m.*, li.item_name as lost_name, li.image_urls as lost_images, li.phone as lost_phone, fi.item_name as found_name, fi.image_urls as found_images, fi.phone as found_phone FROM matches m JOIN items li ON m.lost_item_id = li.id JOIN items fi ON m.found_item_id = fi.id WHERE m.status='pending' ORDER BY m.created_at DESC") # Sorted by created_at DESC
    # Parse for pending_matches
    for pm in pending_matches:
        if pm.get('lost_images'):
            pm['lost_images_list'] = json.loads(pm['lost_images'])
        else:
            pm['lost_images_list'] = []
        if pm.get('found_images'):
            pm['found_images_list'] = json.loads(pm['found_images'])
        else:
            pm['found_images_list'] = []
    return render_template_string(get_base_style("dashboard") + """
<div class="container">
<div class='navbar'>
  <span class='logo'>Lost & Found Portal - Admin</span>
  <span style="display:flex;align-items:center;">
    <a href="{{ url_for('admin_dashboard') }}" class="btn">← Back to Dashboard</a>
  </span>
</div>
<div class='main'>
  <h2>Match Items</h2>
  <div class="pending-matches">
    <h3>Pending Matches ({{ pending_matches|length }}) - Sorted by Date</h3>
    {% for pm in pending_matches %}
    <div class="match-card">
      <h4>Lost: {{ pm['lost_name'] }} | Found: {{ pm['found_name'] }}</h4>
      {% if pm['lost_phone'] %}
      <small>Lost Phone: {{ pm['lost_phone'] }}</small>
      {% endif %}
      {% if pm['found_phone'] %}
      <small>Found Phone: {{ pm['found_phone'] }}</small>
      {% endif %}
      <div class="match-images">
        <div>
          <span class="match-label">Lost Images:</span>
          {% if pm['lost_images_list'] %}
            <div class="images-row">
              {% for url in pm['lost_images_list'][:3] %}
              <img src="{{ url }}" alt="Lost Image" class="thumbnail">
              {% endfor %}
            </div>
          {% endif %}
        </div>
        <div>
          <span class="match-label">Found Images:</span>
          {% if pm['found_images_list'] %}
            <div class="images-row">
              {% for url in pm['found_images_list'][:3] %}
              <img src="{{ url }}" alt="Found Image" class="thumbnail">
              {% endfor %}
            </div>
          {% endif %}
        </div>
      </div>
      <div class="match-actions">
        <a href="{{ url_for('admin_approve_match', match_id=pm['id']) }}" class="btn approve">Approve & Notify Both Parties</a>
        <a href="{{ url_for('admin_reject_match', match_id=pm['id']) }}" class="btn reject">Reject</a>
      </div>
    </div>
    {% endfor %}
  </div>
  <h3>Potential Matches (Fuzzy, Sorted by Score)</h3>
  <div class="card-grid">
    {% for match in potential_matches %}
    <div class="card match">
      <span class="type">Potential (Score: {{ "%.2f"|format(match['score']) }})</span>
      <h4>Lost: {{ match['lost']['item_name'] }}</h4>
      <p>{{ match['lost']['description'] }}</p>
      {% if match['lost']['phone'] %}
      <small>Phone: {{ match['lost']['phone'] }}</small>
      {% endif %}
      {% if match['lost']['image_urls_list'] %}
        <div class="images-row">
          {% for url in match['lost']['image_urls_list'][:3] %}
          <img src="{{ url }}" alt="Lost Image" class="thumbnail">
          {% endfor %}
        </div>
      {% endif %}
      <h4>Found: {{ match['found']['item_name'] }}</h4>
      <p>{{ match['found']['description'] }}</p>
      {% if match['found']['phone'] %}
      <small>Phone: {{ match['found']['phone'] }}</small>
      {% endif %}
      {% if match['found']['image_urls_list'] %}
        <div class="images-row">
          {% for url in match['found']['image_urls_list'][:3] %}
          <img src="{{ url }}" alt="Found Image" class="thumbnail">
          {% endfor %}
        </div>
      {% endif %}
      <form method="POST" action="{{ url_for('admin_create_match') }}" style="margin-top:10px;">
        <input type="hidden" name="lost_id" value="{{ match['lost']['id'] }}">
        <input type="hidden" name="found_id" value="{{ match['found']['id'] }}">
        <button type="submit" class="btn approve">Create Pending Match</button>
      </form>
    </div>
    {% endfor %}
  </div>
  {% if not potential_matches %}
  <p style="text-align:center; color:#666;">No potential matches. Return some found items first.</p>
  {% endif %}
</div>
<div class='footer'>© 2025 Lost & Found Portal (Admin)</div>
</div>
""", potential_matches=potential_matches, pending_matches=pending_matches)
@app.route("/admin_create_match", methods=["POST"])
def admin_create_match():
    if not session.get("admin"):
        return redirect(url_for("admin_login"))
    lost_id = request.form.get("lost_id")
    found_id = request.form.get("found_id")
    if not lost_id or not found_id:
        flash("Invalid match data.")
        return redirect(url_for("admin_matches"))
    try:
        lost_id = int(lost_id)
        found_id = int(found_id)
    except ValueError:
        flash("Invalid IDs provided.")
        return redirect(url_for("admin_matches"))
    if lost_id == 0 or found_id == 0:
        flash("Invalid match IDs.")
        return redirect(url_for("admin_matches"))
    lost = query_db("SELECT * FROM items WHERE id=%s", (lost_id,), fetchone=True)
    found = query_db("SELECT * FROM items WHERE id=%s", (found_id,), fetchone=True)
    if lost and found:
        query_db("""
            INSERT INTO matches (lost_item_id, found_item_id, lost_email, found_email)
            VALUES (%s, %s, %s, %s)
        """, (lost['id'], found['id'], lost['user_email'], found['user_email']), commit=True)
        # Notify admin
        admin_msg = f"New match created: Lost ID {lost['id']} ({lost['item_name']}) matched with Found ID {found['id']} ({found['item_name']}). Please review and notify parties."
        query_db("INSERT INTO notifications (user_email, message) VALUES (%s, %s)", ('admin', admin_msg), commit=True)
    else:
        flash("Lost or found item not found.")
    return redirect(url_for("admin_matches"))
@app.route("/admin_approve_match/<int:match_id>")
def admin_approve_match(match_id):
    if not session.get("admin"):
        return redirect(url_for("admin_login"))
    m = query_db("SELECT * FROM matches WHERE id=%s", (match_id,), fetchone=True)
    if m:
        lost_item = query_db("SELECT * FROM items WHERE id=%s", (m['lost_item_id'],), fetchone=True)
        found_item = query_db("SELECT * FROM items WHERE id=%s", (m['found_item_id'],), fetchone=True)
        if lost_item and found_item:
            query_db("UPDATE items SET status='matched' WHERE id=%s OR id=%s", (m['lost_item_id'], m['found_item_id']), commit=True)
            query_db("UPDATE matches SET status='approved' WHERE id=%s", (match_id,), commit=True)
            loser_email = m['lost_email']
            finder_email = m['found_email']
            # Concise notify loser (includes finder phone)
            loser_subject = "Lost Item Found!"
            finder_phone = found_item.get('phone', 'N/A')
            loser_body = f"""Hi,
'{lost_item['item_name']}' matched & at AIML office.
Desc: {lost_item['description'][:100]}
Found: {found_item['description'][:100]}
Loc: {found_item['location']}
Finder Phone: {finder_phone}
Collect soon.
AIML Lost & Found."""
            ok_loser, info_loser = send_email(loser_email, loser_subject, loser_body)
            # Concise notify finder (includes loser phone)
            finder_subject = "Found Item Matched!"
            loser_phone = lost_item.get('phone', 'N/A')
            finder_body = f"""Hi,
'{found_item['item_name']}' matched with lost item.
Lost: {lost_item['description'][:100]}
Owner Phone: {loser_phone}
Owner notified. Thanks!
AIML Lost & Found."""
            ok_finder, info_finder = send_email(finder_email, finder_subject, finder_body)
            note_loser = f"Match approved and email sent to loser: {info_loser}" if ok_loser else f"Match approved but email to loser failed: {info_loser}"
            note_finder = f"Email sent to finder: {info_finder}" if ok_finder else f"Email to finder failed: {info_finder}"
            query_db("INSERT INTO notifications (user_email, message) VALUES (%s, %s)", (loser_email, note_loser), commit=True)
            query_db("INSERT INTO notifications (user_email, message) VALUES (%s, %s)", (finder_email, note_finder), commit=True)
            # Admin note
            query_db("INSERT INTO notifications (user_email, message) VALUES (%s, %s)", ('admin', f"Match {match_id} approved. Emails sent to both parties."), commit=True)
        else:
            flash("Items not found.")
    return redirect(url_for("admin_matches"))
@app.route("/admin_reject_match/<int:match_id>")
def admin_reject_match(match_id):
    if not session.get("admin"):
        return redirect(url_for("admin_login"))
    m = query_db("SELECT * FROM matches WHERE id=%s", (match_id,), fetchone=True)
    if m:
        query_db("UPDATE matches SET status='rejected' WHERE id=%s", (match_id,), commit=True)
        query_db("INSERT INTO notifications (user_email, message) VALUES (%s, %s)",
                 (m['lost_email'], f"Match rejected by admin for lost item ID {m['lost_item_id']}"), commit=True)
        query_db("INSERT INTO notifications (user_email, message) VALUES (%s, %s)",
                 (m['found_email'], f"Match rejected by admin for found item ID {m['found_item_id']}"), commit=True)
        # Admin note
        query_db("INSERT INTO notifications (user_email, message) VALUES (%s, %s)", ('admin', f"Match {match_id} rejected."), commit=True)
        flash("Match rejected. Notifications sent to parties.")
    return redirect(url_for("admin_matches"))
@app.route("/admin_mark_collected/<int:item_id>")
def admin_mark_collected(item_id):
    if not session.get("admin"):
        return redirect(url_for("admin_login"))
    query_db("UPDATE items SET status='collected' WHERE id=%s", (item_id,), commit=True)
    query_db("UPDATE matches SET status='collected' WHERE found_item_id=%s", (item_id,), commit=True)
    flash("Item marked as collected.")
    return redirect(url_for("admin_dashboard"))
@app.route("/admin_delete/<int:item_id>")
def admin_delete(item_id):
    if not session.get("admin"):
        return redirect(url_for("admin_login"))
    # First, delete related matches to avoid FK constraint
    query_db("DELETE FROM matches WHERE lost_item_id = %s OR found_item_id = %s", (item_id, item_id), commit=True)
    # Then delete the item
    query_db("DELETE FROM items WHERE id=%s", (item_id,), commit=True)
    flash("Item and related matches deleted by admin.")
    return redirect(url_for("admin_full_view"))
@app.route("/admin_logout")
def admin_logout():
    if session.get("admin"):
        pass
    session.pop("admin", None)
    session.pop("admin_username", None)
    return redirect(url_for("admin_login"))
@app.route("/", methods=["GET"])
def root():
    return redirect(url_for("welcome"))
@app.route("/welcome")
def welcome():
    if "email" in session:
        return redirect(url_for("dashboard"))
    return render_template_string(get_base_style("home") + """
<div class="container">
<div class='navbar'>
  <span class='logo'>Lost & Found Portal</span>
  <span>
    <a href="{{ url_for('login') }}" class="btn">Login</a>
    <a href="{{ url_for('register') }}" class="btn">Register</a>
    <a href="{{ url_for('admin_login') }}" class="btn">Admin Login</a>
  </span>
</div>
<div class='main'>
  <h1 style="color:#6a11cb;">Welcome to Lost & Found Portal!</h1>
  <p style="text-align:center; font-size:1.1rem; margin:20px 0;">Track and report lost or found items easily within AIML Department.</p>
</div>
<div class='footer'>© 2025 Lost & Found Portal</div>
</div>
""")
@app.route("/register", methods=["GET", "POST"])
def register():
    if request.method == "POST":
        email = request.form.get("email")
        fn = request.form.get("first_name")
        ln = request.form.get("last_name")
        pw = request.form.get("password")
        if not all([email, fn, ln, pw]):
            flash("All fields required.")
            return render_template_string(get_base_style("auth") + """...""")
        if len(pw) < 6:
            flash("Password must be at least 6 characters.")
            return render_template_string(get_base_style("auth") + """...""")
        pw = generate_password_hash(pw)
        try:
            query_db("INSERT INTO users (email, first_name, last_name, password) VALUES (%s, %s, %s, %s)", (email, fn, ln, pw), commit=True)
            flash("Registered successfully!")
            return redirect(url_for("login"))
        except mysql.connector.IntegrityError:
            flash("Email already registered.")
    return render_template_string(get_base_style("auth") + """
<div class="container">
<div class='navbar'>
  <span class='logo'>Lost & Found Portal</span>
</div>
<div class='main' style='max-width:540px;margin-top:40px;'>
<h2>Register</h2>
<form method="POST" autocomplete="off">
<input name="email" type="email" placeholder="Email" required autocomplete="off"><br>
[01/02, 7:42 pm] Pa 🫶: <input name="first_name" placeholder="First Name" required autocomplete="off"><br>
<input name="last_name" placeholder="Last Name" required autocomplete="off"><br>
<input name="password" type="password" placeholder="Password" required autocomplete="new-password"><br>
<button type="submit" class="btn">Register</button>
</form>
<p>Already have an account? <a href="{{ url_for('login') }}">Login</a></p>
{% with messages = get_flashed_messages() %}{% if messages %}<ul>{% for m in messages %}<li>{{ m }}</li>{% endfor %}</ul>{% endif %}{% endwith %}
</div>
<div class='footer'>© 2025 Lost & Found Portal</div>
</div>
""")
@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        email = request.form.get("email")
        pw = request.form.get("password")
        if not email or not pw:
            flash("Email and password required.")
            return render_template_string(get_base_style("auth") + """...""")
        row = query_db("SELECT * FROM users WHERE email=%s", (email,), fetchone=True)
        if row and check_password_hash(row["password"], pw):
            session.clear() # Clear any existing session (e.g., admin)
            session["email"] = email
            session["first_name"] = row["first_name"]
            return redirect(url_for("dashboard"))
        else:
            flash("Invalid credentials.")
    return render_template_string(get_base_style("auth") + """
<div class="container">
<div class='navbar'>
  <span class='logo'>Lost & Found Portal</span>
</div>
<div class='main' style='max-width:440px;margin-top:40px;'>
<h2>Login</h2>
<form method="POST" autocomplete="off">
<input name="email" type="email" placeholder="Email" required autocomplete="off"><br>
<input name="password" type="password" placeholder="Password" required autocomplete="new-password"><br>
<button type="submit" class="btn">Login</button>
</form>
<p>Don’t have an account? <a href="{{ url_for('register') }}">Register</a></p>
{% with messages = get_flashed_messages() %}{% if messages %}<ul>{% for m in messages %}<li>{{ m }}</li>{% endfor %}</ul>{% endif %}{% endwith %}
</div>
<div class='footer'>© 2025 Lost & Found Portal</div>
</div>
""")
@app.route("/dashboard")
def dashboard():
    if "email" not in session:
        return redirect(url_for("login"))
    items = query_db("SELECT * FROM items ORDER BY id DESC")
    # Parse image_urls for items
    for item in items:
        if item.get('image_urls'):
            item['image_urls_list'] = json.loads(item['image_urls'])
        else:
            item['image_urls_list'] = []
    user_lost = [i for i in items if i['type'] == 'Lost' and i['user_email'] == session["email"]]
    user_found = [i for i in items if i['type'] == 'Found' and i['user_email'] == session["email"]]
    lost_count = len(user_lost)
    found_count = len(user_found)
    user_matches = []
    for item in user_lost:
        matches = query_db("SELECT * FROM matches WHERE lost_item_id=%s ORDER BY created_at DESC", (item['id'],)) # Sorted
        user_matches.extend(matches)
    for item in user_found:
        matches = query_db("SELECT * FROM matches WHERE found_item_id=%s ORDER BY created_at DESC", (item['id'],)) # Sorted
        user_matches.extend(matches)
    # Ensure overall sorting by created_at DESC
    user_matches.sort(key=lambda x: x.get('created_at', ''), reverse=True)
    # Pre-fetch image lists for matches
    for m in user_matches:
        lost_img_row = query_db("SELECT image_urls FROM items WHERE id=%s", (m['lost_item_id'],), fetchone=True)
        m['lost_image_urls_list'] = json.loads(lost_img_row['image_urls']) if lost_img_row and lost_img_row['image_urls'] else []
        found_img_row = query_db("SELECT image_urls FROM items WHERE id=%s", (m['found_item_id'],), fetchone=True)
        m['found_image_urls_list'] = json.loads(found_img_row['image_urls']) if found_img_row and found_img_row['image_urls'] else []
    user_notifications = len(user_matches)
    # NEW: Compute match counts for user's items only (to display in "match column"-like indicator)
    user_item_ids = {i['id'] for i in user_lost + user_found}
    all_matches = query_db("SELECT lost_item_id, found_item_id FROM matches")
    lost_matches_count = {mid: 0 for mid in user_item_ids}
    found_matches_count = {mid: 0 for mid in user_item_ids}
    for m in all_matches:
        if m['lost_item_id'] in user_item_ids:
            lost_matches_count[m['lost_item_id']] += 1
        if m['found_item_id'] in user_item_ids:
            found_matches_count[m['found_item_id']] += 1
    # Assign to items & fetch match details for preview
    for item in items:
        if item['id'] in user_item_ids:
            if item['type'] == 'Lost':
                item['match_count'] = lost_matches_count.get(item['id'], 0)
                # Fetch match preview
                item['matches'] = query_db("SELECT m.status, fi.item_name as found_name, fi.phone as found_phone FROM matches m JOIN items fi ON m.found_item_id = fi.id WHERE m.lost_item_id = %s AND m.status IN ('approved', 'collected') ORDER BY m.created_at DESC LIMIT 1", (item['id'],), fetchone=True) or {}
            else:
                item['match_count'] = found_matches_count.get(item['id'], 0)
                item['matches'] = query_db("SELECT m.status, li.item_name as lost_name, li.phone as lost_phone FROM matches m JOIN items li ON m.lost_item_id = li.id WHERE m.found_item_id = %s AND m.status IN ('approved', 'collected') ORDER BY m.created_at DESC LIMIT 1", (item['id'],), fetchone=True) or {}
        else:
            item['match_count'] = 0 # Non-user items show 0
            item['matches'] = {}
    return render_template_string(get_base_style("dashboard") + """
<div class="container">
<div class='navbar'>
  <span class='logo'>Lost & Found Portal</span>
  <span style="display:flex;align-items:center;">
    <a href="#matches" class="notif" title="View Matches" style="text-decoration:none;">
      🔔
      {% if user_notifications > 0 %}
      <span class="badge">{{ user_notifications }}</span>
      {% endif %}
    </a>
    <span class='user'>Hi, {{ session['first_name'] }} | <a href="{{ url_for('logout') }}" class="btn">Logout</a></span>
  </span>
</div>
<div class='main'>
  <div class="dashboard-header" style="margin-bottom:32px;">
    <h2 style="margin-bottom:0;">Dashboard</h2>
    <span class="college">AIML Department</span>
  </div>
  <div class="card-stats" style="margin-bottom:36px;">
    <div class="stat-card lost">
      <span class="icon">🛑</span>
      <h3>Your Lost</h3>
      <div class="count">{{ lost_count }}</div>
    </div>
    <div class="stat-card found">
      <span class="icon">🔍</span>
      <h3>Your Found</h3>
      <div class="count">{{ found_count }}</div>
    </div>
    <div class="stat-card match">
      <span class="icon">🤝</span>
      <h3>Your Matches</h3>
      <div class="count">{{ user_notifications }}</div>
    </div>
  </div>
  <div style="display:flex; justify-content:center; gap:24px; margin-bottom:32px; flex-wrap:wrap;">
    <a href="{{ url_for('upload', type_='Lost') }}" class="btn" style="background:linear-gradient(90deg,#ff8a65,#ff5252);color:#fff;font-size:1.1rem;box-shadow:0 4px 12px rgba(255,82,82,0.2);">
      <span style="font-size:1.3rem;vertical-align:middle;">➕</span> Report Lost
    </a>
    <a href="{{ url_for('upload', type_='Found') }}" class="btn" style="background:linear-gradient(90deg,#4caf50,#45a049);color:#fff;font-size:1.1rem;box-shadow:0 4px 12px rgba(76,175,80,0.2);">
      <span style="font-size:1.3rem;vertical-align:middle;">🔍</span> Report Found
    </a>
  </div>
  <h3 style="color:#6a11cb; margin-bottom:20px;">All Items</h3>
  <div class="card-grid">
    {% for item in items %}
    <div class="card {% if item['type'] == 'Lost' and item['status'] == 'pending' %}lost pending{% elif item['type'] == 'Lost' %}lost{% elif item['status'] == 'returned' or item['status'] == 'matched' %}found returned{% else %}found{% endif %}">
      <span class="type">{{ item['type'] }} - {{ item['status'] }}</span>
      <div class="edit-delete">
        {% if item['status'] == 'pending' and item['user_email'] == session['email'] %}
          <a href="{{ url_for('edit_item', item_id=item['id']) }}" title="Edit">✏️</a>
        {% endif %}
      </div>
      <strong style="font-size:1.2rem;display:block;margin-bottom:8px;">{{ item['item_name'] }}</strong>
      <p style="margin-bottom:12px;word-break:break-word;line-height:1.4;">{{ item['description'] }}</p>
      <small style="color:#666;">Location: <b>{{ item['location'] }}</b></small>
      <small style="color:#666; display:block; margin-top:4px;">User: <b>{{ item['user_email'] }}</b></small>
      {% if item['match_count'] > 0 and item['matches'] %}
      <div class="match-preview">
        <h5>Match: {{ item['matches']['status'].title() }}</h5>
        <small>{% if item['type'] == 'Lost' %}{{ item['matches']['found_name'] }} (Phone: {{ item['matches']['found_phone'] or 'N/A' }}){% else %}{{ item['matches']['lost_name'] }} (Phone: {{ item['matches']['lost_phone'] or 'N/A' }}){% endif %}</small>
      </div>
      {% elif item['match_count'] > 0 %}
      <small style="color:#388e3c; font-weight:600; display:block; margin-top:4px;">Matches: {{ item['match_count'] }} <span style="background:#e8f5e9; padding:2px 6px; border-radius:4px; font-size:0.8rem;">🤝</span></small>
      {% endif %}
      {% if item['type'] == 'Found' and item['phone'] %}
      <small style="color:#666; display:block; margin-top:4px;">Phone: <b>{{ item['phone'] }}</b></small>
      {% endif %}
      {% if item['image_urls_list'] %}
        <div class="images-row">
          {% for url in item['image_urls_list'] %}
          <img src="{{ url }}" alt="Item Image" class="thumbnail">
          {% endfor %}
        </div>
      {% endif %}
    </div>
    {% endfor %}
  </div>
  {% if not items %}
  <p style="text-align:center; color:#666; padding:40px;">No items reported yet. Start by reporting a lost or found item!</p>
  {% endif %}
  <h3 id="matches" style="color:#6a11cb; margin-top:40px;margin-bottom:20px;">Your Matches (Sorted by Date)</h3>
  <div class="card-grid">
    {% for m in user_matches %}
    <div class="card match {% if m['status'] == 'approved' or m['status'] == 'collected' %}matched{% else %}pending{% endif %}">
      <span class="type">Match Status: {{ m['status'] }}</span>
      <h4 style="color:#388e3c;margin-bottom:8px;">{% if m['lost_email'] == session['email'] %}Your Lost Item Matched!{% else %}Your Found Item Matched!{% endif %}</h4>
      <p style="margin-bottom:12px;">Status: {{ m['status'] }}. Please check with admin if approved for collection.</p>
      <div class="match-images">
        {% if m['lost_image_urls_list'] %}
          <div>
            <span class="match-label">Lost Images:</span>
            <div class="images-row">
              {% for url in m['lost_image_urls_list'][:3] %}
              <img src="{{ url }}" alt="Lost Image" class="thumbnail">
              {% endfor %}
            </div>
          </div>
        {% endif %}
        {% if m['found_image_urls_list'] %}
          <div>
            <span class="match-label">Found Images:</span>
            <div class="images-row">
              {% for url in m['found_image_urls_list'][:3] %}
              <img src="{{ url }}" alt="Found Image" class="thumbnail">
              {% endfor %}
            </div>
          </div>
        {% endif %}
      </div>
    </div>
    {% endfor %}
  </div>
  {% if not user_matches %}
  <p style="text-align:center;color:#666;padding:20px;">No matches yet. Admin will notify you when one is found.</p>
  {% endif %}
</div>
<div class='footer'>© 2025 Lost & Found Portal</div>
</div>
""", items=items, lost_count=lost_count, found_count=found_count, user_matches=user_matches, user_notifications=user_notifications)
@app.route("/logout")
def logout():
    session.clear()
    return redirect(url_for("welcome"))
@app.route("/upload/<type_>", methods=["GET", "POST"])
def upload(type_):
    type = type_ # Avoid keyword conflict
    if "email" not in session:
        return redirect(url_for("login"))
    if request.method == "POST":
        name = request.form.get("item_name")
        desc = request.form.get("description")
        loc = request.form.get("location")
        phone = request.form.get("phone", "") if type == "Found" else None
        if not all([name, desc, loc]):
            flash("Name, description, and location required.")
            return render_template_string(get_base_style("view") + """...""")
        if len(desc) < 10:
            flash("Description too short (min 10 chars).")
            return render_template_string(get_base_style("view") + """...""")
        files = request.files.getlist("images") # Changed to getlist for multiple
        image_urls = []
        for file in files:
            if file and file.filename and file.content_type.startswith('image/'):
                try:
                    res = cloudinary.uploader.upload(file)
                    image_url = res.get("secure_url") or res.get("url")
                    if image_url:
                        image_urls.append(image_url)
                except Exception as e:
                    app.logger.warning(f"Upload failed for {file.filename}: {e}")
        image_urls_json = json.dumps(image_urls) if image_urls else None
        base_columns = "user_email, type, item_name, description, location, image_urls, status"
        base_values = "(%s, %s, %s, %s, %s, %s, %s)"
        base_args = (session["email"], type, name, desc, loc, image_urls_json, 'pending')
        if type == "Found" and phone:
            if not phone.replace('-', '').replace(' ', '').isdigit() or len(phone.replace('-', '').replace(' ', '')) < 10:
                flash("Invalid phone number.")
                return render_template_string(get_base_style("view") + """...""")
            base_columns += ", `phone`"
            base_values = "(%s, %s, %s, %s, %s, %s, %s, %s)"
            base_args = base_args + (phone,)
        query_str = f"INSERT INTO items ({base_columns}) VALUES {base_values}"
        item_id = query_db(query_str, base_args, commit=True, return_id=True)
        if type == "Found":
            subject = "Found Item Reported"
            body = f"""Hi {session['first_name']},
{name} reported. Return to AIML office.
Desc: {desc[:100]}
Loc: {loc}
Phone: {phone or 'N/A'}
Admin review pending.
AIML Lost & Found."""
            send_email(session["email"], subject, body)
            flash("Found item reported! Check your email for return instructions. Admin approval pending.")
        else:
            flash(f"{type} item reported! Admin will handle matching.")
        return redirect(url_for("dashboard"))
    return render_template_string(get_base_style("view") + """
<div class="container">
<div class='navbar'>
  <span class='logo'>Lost & Found Portal</span>
  <span class='user'>Hi, {{ session['first_name'] }}</span>
</div>
<div class='main' style='max-width:600px; padding:16px;'> <!-- Reduced max-width and padding -->
<h2 style='margin-bottom:8px;'>Report {{ type }} Item</h2> <!-- Tighter margin -->
<form method="POST" enctype="multipart/form-data" class="form-compact"> <!-- Compact class -->
  <input name="item_name" placeholder="Item Name" required>
  <textarea name="description" placeholder="Description (details help matching)" required></textarea> <!-- No rows=4, uses min-height -->
  <input name="location" placeholder="Location" required>
  {% if type == 'Found' %}
  <input name="phone" placeholder="Your Phone Number (optional for contact)" type="tel">
  {% endif %}
  <input type="file" name="images" accept="image/*" multiple>
  <button type="submit" class="btn" style='margin-top:4px;'>Submit</button> <!-- Tighter top margin -->
</form>
<a href="{{ url_for('dashboard') }}" class="btn" style='margin-top:8px; display:block;'>Back to Dashboard</a> <!-- Inline block for compactness -->
{% with messages = get_flashed_messages() %}{% if messages %}<ul style='margin:8px 0; padding-left:20px; color:#4caf50;'>{% for m in messages %}<li>{{ m }}</li>{% endfor %}</ul>{% endif %}{% endwith %}
</div>
<div class='footer'>© 2025 Lost & Found Portal</div>
</div>
""", type=type)
@app.route("/edit/<int:item_id>", methods=["GET", "POST"])
def edit_item(item_id):
    if "email" not in session:
        return redirect(url_for("login"))
    item = query_db("SELECT * FROM items WHERE id=%s AND user_email=%s", (item_id, session["email"]), fetchone=True)
    if not item:
        flash("Item not found or unauthorized.")
        return redirect(url_for("dashboard"))
    if item["status"] != "pending":
        flash("Cannot edit non-pending items.")
        return redirect(url_for("dashboard"))
    # Parse image_urls for template
    if item.get('image_urls'):
        item['image_urls_list'] = json.loads(item['image_urls'])
    else:
        item['image_urls_list'] = []
    existing_urls = item['image_urls_list']
    if request.method == "POST":
        name = request.form.get("item_name")
        desc = request.form.get("description")
        loc = request.form.get("location")
        phone = request.form.get("phone", item.get('phone', '')) if item['type'] == 'Found' else None
        if not all([name, desc, loc]):
            flash("Name, description, and location required.")
            return render_template_string(get_base_style("view") + """...""")
        files = request.files.getlist("images") # Multiple files
        new_image_urls = []
        for file in files:
            if file and file.filename and file.content_type.startswith('image/'):
                try:
                    res = cloudinary.uploader.upload(file)
                    new_image_url = res.get("secure_url") or res.get("url")
                    if new_image_url:
                        new_image_urls.append(new_image_url)
                except Exception as e:
                    app.logger.warning(f"Upload failed: {e}")
        # Append new images to existing
        updated_urls = existing_urls + new_image_urls
        image_urls_json = json.dumps(updated_urls) if updated_urls else None
        update_args = (name, desc, loc, image_urls_json, item_id)
        update_str = "UPDATE items SET item_name=%s, description=%s, location=%s, image_urls=%s WHERE id=%s"
        if item['type'] == 'Found' and phone:
            if phone and not phone.replace('-', '').replace(' ', '').isdigit() or len(phone.replace('-', '').replace(' ', '')) < 10:
                flash("Invalid phone number.")
                return render_template_string(get_base_style("view") + """...""")
            update_args = (name, desc, loc, image_urls_json, phone, item_id)
            update_str += ", phone=%s WHERE id=%s"
        query_db(update_str, update_args, commit=True)
        flash("Item updated successfully!")
        return redirect(url_for("dashboard"))
    return render_template_string(get_base_style("view") + """
<div class="container">
<div class='navbar'>
  <span class='logo'>Lost & Found Portal</span>
  <span class='user'>Hi, {{ session['first_name'] }}</span>
</div>
<div class='main' style='max-width:720px;'>
<h2>Edit {{ item['type'] }} Item</h2>
<form method="POST" enctype="multipart/form-data">
<input name="item_name" placeholder="Item Name" value="{{ item['item_name'] }}" required><br>
<textarea name="description" placeholder="Description" required rows="4">{{ item['description'] }}</textarea><br>
<input name="location" placeholder="Location" value="{{ item['location'] }}" required><br>
{% if item['type'] == 'Found' %}
<input name="phone" placeholder="Phone Number" value="{{ item['phone'] or '' }}" type="tel"><br>
{% endif %}
<input type="file" name="images" accept="image/*" multiple><br> <!-- Multiple uploads to append -->
{% if item['image_urls_list'] %}
  <div class="images-row">
    {% for url in item['image_urls_list'] %}
    <img src="{{ url }}" alt="Current Image" class="thumbnail">
    {% endfor %}
  </div>
  <p style="font-size:0.9em; color:#666;">Current images shown above. New uploads will be added.</p>
{% endif %}
<button type="submit" class="btn">Update Item</button>
</form>
<a href="{{ url_for('dashboard') }}" class="btn">Back to Dashboard</a>
{% with messages = get_flashed_messages() %}{% if messages %}<ul>{% for m in messages %}<li>{{ m }}</li>{% endfor %}</ul>{% endif %}{% endwith %}
</div>
<div class='footer'>© 2025 Lost & Found Portal</div>
</div>
""", item=item)
@app.route("/delete/<int:item_id>")
def delete_item(item_id):
    if "email" not in session:
        return redirect(url_for("login"))
    flash("Only admin can delete items.")
    return redirect(url_for("dashboard"))
@app.route("/test_mail")
def test_mail():
    target = session.get("email") or app.config.get("MAIL_USERNAME")
    if not target:
        return "No target email set."
    ok, info = send_email(target, "Test Email", "This is a test email from the Lost & Found app.")
    return f"Email {'sent successfully!' if ok else f'failed: {info}'}"
if __name__ == "__main__":
    app.run(debug=True)
