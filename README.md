import streamlit as st
import pandas as pd
import sqlite3
from datetime import datetime
import os

# --- DATABASE SETUP ---
# Using a local path that Streamlit definitely has write-access to
DB_FILE = "samples_v2.db"

def init_db():
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS samples 
                 (id INTEGER PRIMARY KEY AUTOINCREMENT, 
                  date_received TEXT, 
                  project TEXT, 
                  sample_id TEXT, 
                  location TEXT, 
                  status TEXT,
                  staff TEXT)''')
    conn.commit()
    conn.close()

# --- APP LAYOUT ---
st.set_page_config(page_title="Lab Sample Portal", layout="wide")

# Simple Login Logic
if "authenticated" not in st.session_state:
    st.session_state["authenticated"] = False

if not st.session_state["authenticated"]:
    st.title("🔒 Team Lab Portal")
    pwd = st.text_input("Enter Lab Password", type="password")
    if st.button("Login"):
        if pwd == "LabTeam2026": # This is your password
            st.session_state["authenticated"] = True
            st.rerun()
        else:
            st.error("Incorrect Password")
else:
    # --- LOGGED IN CONTENT ---
    init_db()
    
    st.title("🧪 Sample Reception & Location Portal")
    
    menu = st.sidebar.radio("Navigation", ["Register New Sample", "Search/Locate Samples"])

    if menu == "Register New Sample":
        st.subheader("📥 Register Sample")
        with st.form("reg_form", clear_on_submit=True):
            p_name = st.text_input("Project Name")
            s_id = st.text_input("Sample ID")
            loc = st.text_input("Location (e.g. Fridge 1, Box A)")
            st_name = st.text_input("Your Name")
            status = st.selectbox("Status", ["Received", "In Analysis", "Archived"])
            
            submit = st.form_submit_button("Save to Database")
            
            if submit:
                if p_name and s_id:
                    conn = sqlite3.connect(DB_FILE)
                    c = conn.cursor()
                    dt = datetime.now().strftime("%Y-%m-%d %H:%M")
                    c.execute("INSERT INTO samples (date_received, project, sample_id, location, status, staff) VALUES (?,?,?,?,?,?)",
                              (dt, p_name, s_id, loc, status, st_name))
                    conn.commit()
                    conn.close()
                    st.success(f"Sample {s_id} logged!")
                else:
                    st.error("Project Name and Sample ID are required.")

    elif menu == "Search/Locate Samples":
        st.subheader("🔍 Find a Sample")
        search = st.text_input("Type Project or Sample ID to filter...")
        
        conn = sqlite3.connect(DB_FILE)
        df = pd.read_sql_query("SELECT * FROM samples", conn)
        conn.close()
        
        if search:
            df = df[df['project'].str.contains(search, case=False) | df['sample_id'].str.contains(search, case=False)]
        
        st.dataframe(df, use_container_width=True)
