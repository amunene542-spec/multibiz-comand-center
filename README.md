import sqlite3
import streamlit as st
import pandas as pd
from datetime import date, timedelta
import urllib.parse
from io import BytesIO
from fpdf import FPDF

# --- DATABASE INITIALIZATION ---
def init_db():
    conn = sqlite3.connect("business_management.db")
    c = conn.cursor()
    
    # Dairy Herd
    c.execute('''CREATE TABLE IF NOT EXISTS dairy (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    tag_id TEXT,
                    breed TEXT,
                    status TEXT)''')

    # Daily Milk Logs
    c.execute('''CREATE TABLE IF NOT EXISTS milk_logs (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    tag_id TEXT,
                    am_liters REAL,
                    pm_liters REAL,
                    log_date TEXT)''')

    # Breeding & Vet Health Records
    c.execute('''CREATE TABLE IF NOT EXISTS vet_breeding (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    tag_id TEXT,
                    record_type TEXT,
                    event_date TEXT,
                    expected_date TEXT,
                    notes TEXT)''')

    # Crop Operations & ROI
    c.execute('''CREATE TABLE IF NOT EXISTS crops (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    crop_name TEXT,
                    plot_location TEXT,
                    planting_date TEXT,
                    expected_harvest TEXT,
                    status TEXT,
                    input_cost REAL DEFAULT 0.0,
                    harvest_revenue REAL DEFAULT 0.0)''')

    # Electronics & Store Inventory
    c.execute('''CREATE TABLE IF NOT EXISTS electronics (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    item_name TEXT,
                    category TEXT,
                    cost_price REAL,
                    selling_price REAL,
                    stock INTEGER)''')

    # Point of Sale Transactions
    c.execute('''CREATE TABLE IF NOT EXISTS sales (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    item_name TEXT,
                    quantity INTEGER,
                    total_amount REAL,
                    mpesa_code TEXT,
                    sale_date TEXT)''')

    # Customer Credit / Layaway Tracker
    c.execute('''CREATE TABLE IF NOT EXISTS customer_credit (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    customer_name TEXT,
                    phone TEXT,
                    item_description TEXT,
                    total_credit REAL,
                    amount_paid REAL,
                    balance_due REAL,
                    due_date TEXT,
                    status TEXT)''')

    # Device Repair Job Cards & Warranties
    c.execute('''CREATE TABLE IF NOT EXISTS repairs (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    customer_name TEXT,
                    phone TEXT,
                    device_model TEXT,
                    issue TEXT,
                    status TEXT,
                    cost REAL,
                    warranty_expiry TEXT,
                    entry_date TEXT)''')

    # Fleet, Transport & Delivery Logs
    c.execute('''CREATE TABLE IF NOT EXISTS fleet (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    vehicle_name TEXT,
                    status TEXT,
                    current_mileage REAL,
                    next_service_km REAL,
                    last_service_date TEXT)''')

    c.execute('''CREATE TABLE IF NOT EXISTS fuel_logs (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    vehicle_name TEXT,
                    liters REAL,
                    cost REAL,
                    mileage_km REAL,
                    log_date TEXT)''')

    c.execute('''CREATE TABLE IF NOT EXISTS transport_trips (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    vehicle_name TEXT,
                    destination TEXT,
                    cargo TEXT,
                    fare_collected REAL,
                    log_date TEXT)''')

    # Caretaker & Property Maintenance Tasks
    c.execute('''CREATE TABLE IF NOT EXISTS caretaker_tasks (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    task_name TEXT,
                    category TEXT,
                    status TEXT,
                    notes TEXT,
                    task_date TEXT)''')

    # Feed Formulation Logs
    c.execute('''CREATE TABLE IF NOT EXISTS feed_batches (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    batch_name TEXT,
                    maize_kg REAL,
                    wheat_kg REAL,
                    omena_kg REAL,
                    total_cost REAL,
                    log_date TEXT)''')

    # General Accounting Ledger & Venture Tracking
    c.execute('''CREATE TABLE IF NOT EXISTS ledger (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    venture TEXT,
                    type TEXT,
                    category TEXT,
                    amount REAL,
                    description TEXT,
                    record_date TEXT)''')

    # Cash & M-Pesa Reconciliation Table
    c.execute('''CREATE TABLE IF NOT EXISTS cash_reconciliation (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    recon_date TEXT,
                    expected_cash REAL,
                    actual_cash REAL,
                    expected_mpesa REAL,
                    actual_mpesa REAL,
                    notes TEXT)''')

    conn.commit()
    conn.close()

init_db()

def get_connection():
    return sqlite3.connect("business_management.db")

# --- PDF RECEIPT GENERATOR FUNCTION ---
def create_pdf_receipt(receipt_title, customer_name, item_details, amount, mpesa_code, date_str):
    pdf = FPDF()
    pdf.add_page()
    pdf.set_font("Arial", "B", 16)
    pdf.cell(200, 10, txt="ANTONY TECH SOLUTIONS", ln=True, align="C")
    pdf.set_font("Arial", "", 10)
    pdf.cell(200, 6, txt="Meru, Kenya | Phone: 0712345678", ln=True, align="C")
    pdf.line(10, 25, 200, 25)
    
    pdf.ln(10)
    pdf.set_font("Arial", "B", 12)
    pdf.cell(200, 8, txt=receipt_title, ln=True, align="L")
    
    pdf.set_font("Arial", "", 11)
    pdf.cell(200, 6, txt=f"Date: {date_str}", ln=True, align="L")
    pdf.cell(200, 6, txt=f"Customer Name: {customer_name}", ln=True, align="L")
    pdf.cell(200, 6, txt=f"Details: {item_details}", ln=True, align="L")
    pdf.cell(200, 6, txt=f"Total Amount: KSh {amount:,.2f}", ln=True, align="L")
    pdf.cell(200, 6, txt=f"Payment Ref: {mpesa_code if mpesa_code else 'Cash Payment'}", ln=True, align="L")
    
    pdf.ln(15)
    pdf.set_font("Arial", "I", 9)
    pdf.cell(200, 6, txt="Thank you for doing business with us! Goods once sold are not returnable.", ln=True, align="C")
    
    pdf_output = pdf.output(dest='S')
    if isinstance(pdf_output, str):
        pdf_output = pdf_output.encode('latin1')
    return BytesIO(pdf_output)

# --- APP CONFIGURATION ---
st.set_page_config(page_title="MultiBiz Command Center Pro", layout="wide")

# --- SECURE AUTHENTICATION ---
ADMIN_USER = st.secrets.get("ADMIN_USER", "admin")
ADMIN_PASS = st.secrets.get("ADMIN_PASS", "admin123")
SHOP_USER = st.secrets.get("SHOP_USER", "shop")
SHOP_PASS = st.secrets.get("SHOP_PASS", "shop123")

if "logged_in" not in st.session_state:
    st.session_state.logged_in = False
    st.session_state.role = None
    st.session_state.username = ""

if not st.session_state.logged_in:
    st.title("🔐 MultiBiz Secure Login Portal")
    st.caption("Enter your credentials to access your assigned command center modules.")
    
    with st.form("login_form"):
        username = st.text_input("Username")
        password = st.text_input("Password", type="password")
        submit_login = st.form_submit_button("Log In")
        
        if submit_login:
            if username == ADMIN_USER and password == ADMIN_PASS:
                st.session_state.logged_in = True
                st.session_state.role = "Admin"
                st.session_state.username = "Anthony (Master Admin)"
                st.success("Welcome back, Boss!")
                st.rerun()
            elif username == SHOP_USER and password == SHOP_PASS:
                st.session_state.logged_in = True
                st.session_state.role = "Shop Staff"
                st.session_state.username = "Electronics Shop Staff"
                st.success("Logged in successfully!")
                st.rerun()
            else:
                st.error("Invalid username or password.")
    st.stop()

st.sidebar.write(f"👤 Logged in as: **{st.session_state.username}**")
if st.sidebar.button("Log Out"):
    st.session_state.logged_in = False
    st.session_state.role = None
    st.rerun()

st.title("💼 MultiBiz Command Center Pro")

# --- NAVIGATION MENU ---
if st.session_state.role == "Admin":
    menu = [
        "Dashboard & Analytics", 
        "Daily Quick-Log & WhatsApp Digest",
        "Dairy Herd & Breeding", 
        "Crop Operations & ROI", 
        "Feed Formulation",
        "Electronics POS, Credit & Repairs", 
        "Transport, Fuel & Deliveries",
        "Caretaker Tasks & Patrols",
        "Inventory Management", 
        "Bookkeeping & Reconciliation"
    ]
else:
    menu = [
        "Electronics POS, Credit & Repairs", 
        "Inventory Management"
    ]

choice = st.sidebar.selectbox("Navigation", menu)

# --- 1. DASHBOARD & ANALYTICS ---
if choice == "Dashboard & Analytics":
    st.subheader("📊 Executive Summary, Financial Breakdown & Visual Analytics")
    
    venture_filter = st.selectbox("Filter Financials by Venture", ["All Ventures", "Dairy Farm", "Crops", "Electronics Shop", "Transport & Fleet", "Property Management"])

    conn = get_connection()
    c = conn.cursor()

    c.execute("SELECT COUNT(*) FROM dairy")
    cows_count = c.fetchone()[0] or 0

    c.execute("SELECT SUM(am_liters + pm_liters) FROM milk_logs WHERE log_date = ?", (str(date.today()),))
    today_milk = c.fetchone()[0] or 0.0

    if venture_filter == "All Ventures":
        c.execute("SELECT SUM(total_amount) FROM sales")
        sales_rev = c.fetchone()[0] or 0.0
        c.execute("SELECT SUM(cost) FROM repairs WHERE status='Completed'")
        repair_rev = c.fetchone()[0] or 0.0
        c.execute("SELECT SUM(amount) FROM ledger WHERE type='Income'")
        other_inc = c.fetchone()[0] or 0.0
        total_revenue = sales_rev + repair_rev + other_inc
        c.execute("SELECT SUM(amount) FROM ledger WHERE type='Expense'")
        expense_total = c.fetchone()[0] or 0.0
    else:
        c.execute("SELECT SUM(amount) FROM ledger WHERE type='Income' AND venture=?", (venture_filter,))
        total_revenue = c.fetchone()[0] or 0.0
        c.execute("SELECT SUM(amount) FROM ledger WHERE type='Expense' AND venture=?", (venture_filter,))
        expense_total = c.fetchone()[0] or 0.0

    c.execute("SELECT item_name, stock FROM electronics WHERE stock <= 3")
    low_stock_items = c.fetchall()
    conn.close()

    net_profit = total_revenue - expense_total

    col1, col2, col3, col4, col5 = st.columns(5)
    col1.metric("Dairy Herd", f"{cows_count} Head")
    col2.metric("Today's Milk Yield", f"{today_milk:.1f} L")
    col3.metric("Selected Revenue", f"KSh {total_revenue:,.2f}")
    col4.metric("Selected Expenses", f"KSh {expense_total:,.2f}")
    col5.metric("Net Profit / Loss", f"KSh {net_profit:,.2f}")

# --- 2. DAILY QUICK-LOG & WHATSAPP DIGEST ---
elif choice == "Daily Quick-Log & WhatsApp Digest":
    st.subheader("⚡ Unified Daily Operations & Automated WhatsApp Briefing")
    with st.form("quick_log_form"):
        log_date_val = st.date_input("Log Date", value=date.today())
        q_patrol_notes = st.text_input("Patrol Notes / Observations", value="All secure, perimeter fence and water tanks checked.")
        submit_quick = st.form_submit_button("Save Daily Entries")
        if submit_quick:
            conn = get_connection()
            c = conn.cursor()
            c.execute("INSERT INTO caretaker_tasks (task_name, category, status, notes, task_date) VALUES (?, ?, ?, ?, ?)",
                      ("Daily Night Patrol & Security Check", "Security / Patrol", "Completed", q_patrol_notes, str(log_date_val)))
            conn.commit()
            conn.close()
            st.success("✅ Daily entries saved successfully!")

# --- 3. DAIRY HERD & BREEDING ---
elif choice == "Dairy Herd & Breeding":
    st.subheader("🐮 Dairy Herd & Milk Production")
    st.info("Manage your dairy herd registration and daily milk yield.")

# --- 4. CROP OPERATIONS & ROI ---
elif choice == "Crop Operations & ROI":
    st.subheader("🌾 Crop Plot Management & Profitability Tracker")
    st.info("Track input expenses and harvest revenue for garlic, maize, passion fruit, etc.")

# --- 5. FEED FORMULATION CALCULATOR ---
elif choice == "Feed Formulation":
    st.subheader("🐓 Custom Feed Formulation Calculator")
    st.info("Calculate feed batches using maize, wheat, and omena.")

# --- 6. ELECTRONICS POS, CREDIT & REPAIRS ---
elif choice == "Electronics POS, Credit & Repairs":
    st.subheader("🛒 Supermarket Checkout & Repair Till")
    
    tab_pos, tab_credit, tab_repair = st.tabs(["Fast Till (POS)", "Customer Credit (Denzi)", "Repair Job Cards"])

    with tab_pos:
        conn = get_connection()
        c = conn.cursor()
        c.execute("SELECT item_name, selling_price, stock FROM electronics WHERE stock > 0")
        available_items = c.fetchall()
        conn.close()

        if available_items:
            with st.form("pos_supermarket_form"):
                item_names = [item[0] for item in available_items]
                selected_item = st.selectbox("Search / Select Item", item_names)
                
                item_detail = next(item for item in available_items if item[0] == selected_item)
                unit_price, max_stock = item_detail[1], item_detail[2]
                
                st.info(f"Unit Price: KSh {unit_price:,.2f} | Available Stock: **{max_stock} units**")

                qty = st.number_input("Quantity", min_value=1, max_value=max_stock, step=1, value=1)
                total_price = qty * unit_price
                st.markdown(f"### 💰 Total Due: KSh {total_price:,.2f}")
                
                cust_name = st.text_input("Customer Name", value="Walk-in Customer")
                payment_method = st.selectbox("Payment Method", ["Cash", "M-Pesa"])
                mpesa_code = st.text_input("M-Pesa Confirmation Code (Required if paid via M-Pesa)")

                submit_pos = st.form_submit_button("🛒 Complete Checkout & Generate Receipt")

                if submit_pos:
                    if payment_method == "M-Pesa" and not mpesa_code:
                        st.error("Please enter the M-Pesa confirmation code.")
                    else:
                        code_label = mpesa_code if payment_method == "M-Pesa" else "Cash Payment"
                        
                        conn = get_connection()
                        c = conn.cursor()
                        c.execute("UPDATE electronics SET stock = stock - ? WHERE item_name = ?", (qty, selected_item))
                        c.execute("INSERT INTO sales (item_name, quantity, total_amount, mpesa_code, sale_date) VALUES (?, ?, ?, ?, ?)",
                                  (selected_item, qty, total_price, code_label, str(date.today())))
                        c.execute("INSERT INTO ledger (venture, type, category, amount, description, record_date) VALUES (?, ?, ?, ?, ?, ?)",
                                  ("Electronics Shop", "Income", "Retail Sale", total_price, f"Sold {qty} x {selected_item} via {payment_method} ({code_label})", str(date.today())))
                        conn.commit()
                        conn.close()
                        
                        st.success(f"✅ Sale completed successfully! Total: KSh {total_price:,.2f}")

                        pdf_file = create_pdf_receipt("SUPERMARKET SALES RECEIPT", cust_name, f"{qty} x {selected_item}", total_price, code_label, str(date.today()))
                        st.download_button(
                            label="📥 Download / Print Official PDF Receipt",
                            data=pdf_file,
                            file_name=f"receipt_{cust_name}_{date.today()}.pdf",
                            mime="application/pdf"
                        )
        else:
            st.warning("⚠️ No stock available. Please add items in 'Inventory Management' first.")

    with tab_credit:
        st.markdown("#### 📝 Customer Credit / Layaway Tracker")
        with st.form("credit_form"):
            c_name = st.text_input("Customer Name")
            c_phone = st.text_input("Customer Phone Number")
            c_item = st.text_input("Item Description")
            tot_cred = st.number_input("Total Credit Amount (KSh)", min_value=0.0, format="%.2f")
            amt_paid = st.number_input("Deposit Paid (KSh)", min_value=0.0, format="%.2f")
            due_dt = st.date_input("Due Date", value=date.today() + timedelta(days=30))
            
            submit_credit = st.form_submit_button("Save Credit Record")
            if submit_credit and c_name and tot_cred > 0:
                bal_due = tot_cred - amt_paid
                conn = get_connection()
                c = conn.cursor()
                c.execute("INSERT INTO customer_credit (customer_name, phone, item_description, total_credit, amount_paid, balance_due, due_date, status) VALUES (?, ?, ?, ?, ?, ?, ?, ?)",
                          (c_name, c_phone, c_item, tot_cred, amt_paid, bal_due, str(due_dt), "Active Balance"))
                conn.commit()
                conn.close()
                st.success(f"Credit saved for {c_name}. Balance Due: KSh {bal_due:,.2f}")

    with tab_repair:
        st.markdown("#### 🛠️ Device Repair Job Card")
        with st.form("repair_form"):
            r_name = st.text_input("Customer Name")
            r_phone = st.text_input("Customer Phone Number")
            dev_model = st.text_input("Device Model (e.g., Samsung A16)")
            issue_desc = st.text_area("Issue Description")
            rep_cost = st.number_input("Agreed Fee (KSh)", min_value=0.0)
            rep_status = st.selectbox("Status", ["Received / Pending", "In Progress", "Ready for Pickup", "Completed"])
            
            submit_rep = st.form_submit_button("Save Job Card")
            if submit_rep and r_name:
                conn = get_connection()
                c = conn.cursor()
                c.execute("INSERT INTO repairs (customer_name, phone, device_model, issue, status, cost, entry_date) VALUES (?, ?, ?, ?, ?, ?, ?)",
                          (r_name, r_phone, dev_model, issue_desc, rep_status, rep_cost, str(date.today())))
                conn.commit()
                conn.close()
                st.success(f"Job card created for {r_name}.")

# --- 7. TRANSPORT, FUEL & DELIVERIES ---
elif choice == "Transport, Fuel & Deliveries":
    st.subheader("🚚 Fleet & Delivery Logs")
    st.info("Manage your 150cc Captain motorcycle trips, deliveries, and fuel expenses.")

# --- 8. CARETAKER TASKS & PATROLS ---
elif choice == "Caretaker Tasks & Patrols":
    st.subheader("📋 Caretaker & Night Patrol Log")
    st.info("Log your security checks and property maintenance tasks.")

# --- 9. INVENTORY MANAGEMENT ---
elif choice == "Inventory Management":
    st.subheader("📦 Inventory Stock Management")
    with st.form("inv_form"):
        item_name = st.text_input("Item Name")
        category = st.text_input("Category")
        cost_price = st.number_input("Cost Price (KSh)", min_value=0.0)
        selling_price = st.number_input("Selling Price (KSh)", min_value=0.0)
        stock = st.number_input("Initial Stock Quantity", min_value=0, step=1)
        submit = st.form_submit_button("Save Item")
        if submit and item_name:
            conn = get_connection()
            c = conn.cursor()
            c.execute("INSERT INTO electronics (item_name, category, cost_price, selling_price, stock) VALUES (?, ?, ?, ?, ?)",
                      (item_name, category, cost_price, selling_price, stock))
            conn.commit()
            conn.close()
            st.success(f"Added {item_name} to stock!")
            st.rerun()

    conn = get_connection()
    inv_df = pd.read_sql_query("SELECT item_name, category, selling_price, stock FROM electronics", conn)
    conn.close()
    if not inv_df.empty:
        st.dataframe(inv_df, use_container_width=True)

# --- 10. BOOKKEEPING & RECONCILIATION ---
elif choice == "Bookkeeping & Reconciliation":
    st.subheader("💰 Financial Ledger & Shift Reconciliation")

    tab_led, tab_recon, tab_exp = st.tabs(["Record Transaction", "Shift Cash & M-Pesa Reconciliation", "Export Excel Backups"])

    with tab_led:
        with st.form("ledger_form"):
            venture = st.selectbox("Select Business Venture", ["Dairy Farm", "Crops", "Electronics Shop", "Transport & Fleet", "Property Management", "General / Personal"])
            record_type = st.selectbox("Record Type", ["Expense", "Income", "Investment"])
            category = st.text_input("Category (e.g., Feed, Fertilizer, Fuel, Capital)")
            amount = st.number_input("Amount (KSh)", min_value=0.0, format="%.2f")
            description = st.text_area("Notes / Details")
            submit = st.form_submit_button("Record Transaction")

            if submit and amount > 0:
                conn = get_connection()
                c = conn.cursor()
                c.execute("INSERT INTO ledger (venture, type, category, amount, description, record_date) VALUES (?, ?, ?, ?, ?, ?)",
                          (venture, record_type, category, amount, description, str(date.today())))
                conn.commit()
                conn.close()
                st.success("Ledger entry saved!")

        st.markdown("### Financial Ledger History")
        conn = get_connection()
        ledger_df = pd.read_sql_query("SELECT venture, type, category, amount, description, record_date FROM ledger ORDER BY id DESC", conn)
        conn.close()
        if not ledger_df.empty:
            st.dataframe(ledger_df, use_container_width=True)

    with tab_recon:
        st.markdown("#### 💵 End-of-Day Shift Reconciliation (Cash Drawer & M-Pesa Till)")
        with st.form("recon_form"):
            recon_date = st.date_input("Reconciliation Date", value=date.today())
            exp_cash = st.number_input("Expected Cash Sales (KSh)", min_value=0.0, format="%.2f")
            act_cash = st.number_input("Actual Cash in Drawer (KSh)", min_value=0.0, format="%.2f")
            exp_mpesa = st.number_input("Expected M-Pesa Sales (KSh)", min_value=0.0, format="%.2f")
            act_mpesa = st.number_input("Actual M-Pesa Till Balance (KSh)", min_value=0.0, format="%.2f")
            recon_notes = st.text_area("Variance Notes / Remarks")

            submit_recon = st.form_submit_button("Save Shift Reconciliation")

            if submit_recon:
                cash_diff = act_cash - exp_cash
                mpesa_diff = act_mpesa - exp_mpesa
                
                conn = get_connection()
                c = conn.cursor()
                c.execute("INSERT INTO cash_reconciliation (recon_date, expected_cash, actual_cash, expected_mpesa, actual_mpesa, notes) VALUES (?, ?, ?, ?, ?, ?)",
                          (str(recon_date), exp_cash, act_cash, exp_mpesa, act_mpesa, f"Cash Diff: {cash_diff:,.2f} | M-Pesa Diff: {mpesa_diff:,.2f} | {recon_notes}"))
                conn.commit()
                conn.close()

                st.success("✅ Shift reconciliation saved!")

        st.markdown("### Past Reconciliation Reports")
        conn = get_connection()
        recon_df = pd.read_sql_query("SELECT recon_date, expected_cash, actual_cash, expected_mpesa, actual_mpesa, notes FROM cash_reconciliation ORDER BY id DESC", conn)
        conn.close()
        if not recon_df.empty:
            st.dataframe(recon_df, use_container_width=True)

    with tab_exp:
        st.markdown("#### 📥 One-Click Excel / CSV Data Backups")
        conn = get_connection()
        ledger_df = pd.read_sql_query("SELECT * FROM ledger", conn)
        conn.close()
        if not ledger_df.empty:
            st.download_button(
                label="Download Financial Ledger (CSV)",
                data=ledger_df.to_csv(index=False).encode('utf-8'),
                file_name=f"financial_ledger_backup_{date.today()}.csv",
                mime="text/csv"
            )
