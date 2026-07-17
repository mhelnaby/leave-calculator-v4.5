# Leave Balance Management System - Code Documentation & Deep Dive

This document provides a comprehensive, line-by-line technical review of the **Leave Balance Management System** implemented in `leave_calculator.py`. This system is built using **FastAPI**, **Pandas**, and **SQLAlchemy ORM (SQLite)** to deliver a reliable, high-performance, and compliant leave-tracking solution.

---

## 1. Imports, Logging & Global Initialization

```python
import sys, os, glob, threading, time, io
import warnings
warnings.filterwarnings('ignore')
import logging
logging.basicConfig(level=logging.INFO)

from datetime import date, timedelta, datetime
import pandas as pd
System & File Operations: Imports basic Python standard libraries (sys, os, io) to manage platform-independent paths and process in-memory byte streams.

Warning Suppression: Suppresses non-critical warnings (warnings.filterwarnings('ignore')) to keep the terminal console output clean and production-ready.

Logging: Initializes Python's standard logging at the INFO level to monitor real-time system executions and pinpoint runtime issues.

Data & Time Management: Imports datetime objects for strict calendar operations and pandas to carry out high-speed, vectorized data frame computations.

2. Database Connection & Environment Adaptability
Python
from sqlalchemy import create_engine, Column, String, Integer, text
from sqlalchemy.orm import sessionmaker, declarative_base
from sqlalchemy.dialects.sqlite import insert as sqlite_upsert

from fastapi import FastAPI, Request, UploadFile, File, Form
from fastapi.responses import HTMLResponse, JSONResponse
import uvicorn
SQLAlchemy ORM & SQLite: Imports the required engines and schema classes to map Python objects directly to database tables.

FastAPI Ecosystem: Loads web server utilities, HTTP request handlers, and multi-part form processors to handle Excel/CSV uploads seamlessly.

Python
PORT = 8080

if getattr(sys, 'frozen', False):
    BASE_DIR = os.path.dirname(sys.executable)
else:
    BASE_DIR = os.path.dirname(os.path.abspath(__file__))

DB_PATH = os.path.join(BASE_DIR, "leave_balance.db")
DATABASE_URL = f"sqlite:///{DB_PATH}"

engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()
Executable Deployment Guard: The sys.frozen check detects if the application is running as a compiled executable (e.g., packaged with PyInstaller). If frozen, it dynamically resolves the directory path to ensure the database file (leave_balance.db) is generated in the executable's directory rather than a temporary directory.

Multi-Threading Support: The configuration check_same_thread: False permits FastAPI's asynchronous event loops to execute database transactions across parallel worker threads safely.

3. Database Table Schemas (The Blueprint)
These models define how the application data is structured within SQLite:

Python
class Employee(Base):
    __tablename__ = "employees"
    uae_id = Column(String, primary_key=True, index=True)
    acd_id = Column(String)
    Name = Column(String)
    Title = Column(String)
    Queue = Column(String)
    hiring_date = Column(String)
    lwd = Column(String)  # Last Working Day
    certification_date = Column(String)
    LOB = Column(String)
    S_LOB = Column(String)
Employee Profiles Table: Stores metadata for all employees. The lwd (Last Working Day) column acts as a ceiling for active leave accrual; if populated, the system automatically cuts off calculations at that date.

Python
class Attendance(Base):
    __tablename__ = "attendance"
    uae_id = Column(String, primary_key=True)
    Date = Column(String, primary_key=True)  # Composite Primary Key
    acd_id = Column(String)
    date_of_join = Column(String)
    go_live_date = Column(String)
    final_status = Column(String)
    final_status_clean = Column(String)
    queue = Column(String)
    year = Column(Integer)
    month = Column(Integer)
Composite Primary Key Integrity: By binding uae_id and Date together as a Composite Primary Key, the database physically rejects any duplicate attendance entries for the same employee on the same date. This prevents corrupted or doubled data records.

Python
class Holiday(Base):
    __tablename__ = "holidays"
    Holiday_Date = Column(String, primary_key=True)
    Holiday_Name = Column(String)

class NotEligibleShift(Base):
    __tablename__ = "not_eligible_shifts"
    shift_name = Column(String, primary_key=True)

class Disability(Base):
    __tablename__ = "disability"
    uae_id = Column(String, primary_key=True)
    start_date = Column(String)
Exclusions & Accelerated Tracks: These lookup tables manage national holidays, shifts ineligible for holiday compensations, and registered employees with disabilities who are eligible for higher monthly leave accrual rates.

Python
Base.metadata.create_all(bind=engine)
Automatic Migrations: On startup, this command scans SQLite and automatically generates any tables that are missing, eliminating the need for manual database setup.

4. System Business Logic Constants
Python
CUTOFF_DATE = date.today()
CURRENT_YEAR = CUTOFF_DATE.year

LAW_2025_EFFECTIVE_DATE = pd.Timestamp('2025-09-01')

LEAVE_TYPE_WEIGHTS = {
    "annual": 1.0, 
    "half annual": 0.5, 
    "halfday": 0.5,           # Added: Half-day deduction support (0.5)
    "casual": 1.0, 
    "annual exception": 1.0
}

BUILT_IN_EXCLUDED_HOLIDAYS = {
    'off', 'sick', 'casual', 'absent', 'no show', 'comp', 
    'unpaid', 'maternity', 'bereavement', 'sick leave', 
    'casual leave', 'unpaid leave', 'vto', 'absenteeism'
}
Deduction Weights: Assigns fractional costs to different leave categories. halfday and half annual correctly deduct exactly 0.5 of a day from the annual balance, preventing the system from penalizing employees a full day for partial absences.

Labor Law Threshold: Holds the constant date of the updated 2025 Labor Law implementation (2025-09-01) used to dynamically branch calculations.

5. The Calculation Engine (load_all_data)
This core routine loads, cleans, processes, and joins SQL datasets entirely in RAM to compute complex parameters across all active profiles.

Python
def load_all_data():
    db = SessionLocal()
    try:
        # 1. Fetch Disability Registry
        dis_list = db.query(Disability).all()
        disability_dict = {
            str(d.uae_id).strip().lower(): pd.to_datetime(d.start_date, errors='coerce') 
            for d in dis_list if d.start_date
        }
Defensive ID Matching: All incoming IDs are processed with .strip().lower() to neutralize human entry errors, such as trailing spaces ("Casual " vs "Casual"), which would otherwise corrupt comparison checks.

Python
        # 2. Extract the actual Go-Live Dates directly from the attendance logs
        go_live_query = db.execute(text("""
            SELECT uae_id, MAX(go_live_date) as actual_go_live 
            FROM attendance 
            WHERE go_live_date IS NOT NULL AND go_live_date != '' 
            GROUP BY uae_id
        """)).fetchall()
        go_live_dict = {row[0]: pd.to_datetime(row[1], errors='coerce') for row in go_live_query}
Dynamic Go-Live Detection: Extracts the maximum (most recent) go_live entry found within daily attendance logs, preventing the need for manual record syncing.

Monthly Accrual Engine (Pro-rata Law Interpreter)
Python
    def calc_monthly_accruals(row):
        hiring_date = row['hiring_date']
        calc_date = row['calc_date']
        if pd.isna(hiring_date) or pd.isna(calc_date) or hiring_date >= calc_date: 
            return 0.0
        
        is_disability = (str(row['uae_id']).strip().lower() in disability_dict)
        dis_start = disability_dict.get(str(row['uae_id']).strip().lower(), pd.NaT)
        total_balance = 0.0
        current_marker = pd.Timestamp(hiring_date.year, hiring_date.month, 1)
        end_marker = pd.Timestamp(calc_date.year, calc_date.month, 1)
        
        while current_marker <= end_marker:
            service_months = ((current_marker - hiring_date).days) / 30.4375
            
            # Condition A: Disability Registry Acceleration (3.75 days per month)
            if is_disability and pd.notna(dis_start) and current_marker >= dis_start:
                monthly_rate = 3.75
            # Condition B: Pre-2025 Labor Law Rules
            elif hiring_date < LAW_2025_EFFECTIVE_DATE:
                monthly_rate = 2.50 if service_months >= 120.0 else 1.75
            # Condition C: Post-September 2025 Labor Law Rules (Graduated Scale)
            else:
                if service_months < 12.0: 
                    monthly_rate = 1.25  # First year limitation under new law
                else: 
                    monthly_rate = 2.50 if service_months >= 120.0 else 1.75
Disability Track (Condition A): Accelerated track awards 3.75 days per month starting from the disability start date.

Pre-2025 Law (Condition B): Uses standard rates of 1.75 or 2.50 days per month depending on tenure (under or over 10 years of service).

Post-2025 Law (Condition C): Restricts the first-year rate to 1.25 days per month, scaling up to standard rates afterward.

Python
            # Apply fractional ratios for the starting and ending months (Pro-rata)
            if current_marker.year == hiring_date.year and current_marker.month == hiring_date.month:
                days_in_month = pd.Period(hiring_date, freq='M').days_in_month
                fraction = (days_in_month - hiring_date.day + 1) / days_in_month
                total_balance += monthly_rate * fraction
            elif current_marker.year == calc_date.year and current_marker.month == calc_date.month:
                days_in_month = pd.Period(calc_date, freq='M').days_in_month
                fraction = calc_date.day / days_in_month
                total_balance += monthly_rate * fraction
            else:
                total_balance += monthly_rate
Pro-rata Calculations: Calculates partial leave accruals for the hire month and departure month. For example, if hired on the 15th, the system awards a precise fraction of the month's accrual based on the exact remaining calendar days.

Used Leaves Calculation
Python
    leave_mask = attendance['final_status_clean'].isin(LEAVE_TYPE_WEIGHTS.keys())
    leave_days = attendance[leave_mask].copy()
    if not leave_days.empty:
        leave_days['day_weight'] = leave_days['final_status_clean'].map(LEAVE_TYPE_WEIGHTS)
        leave_days = leave_days.merge(holidays, left_on='Date', right_on='Holiday_Date', how='left')
        leave_days = leave_days.merge(employees[['uae_id', 'go_live']], on='uae_id', how='left')
        
        # Avoid deducting annual balance if the booked leave coincides with a public holiday
        leave_days['exclude'] = (
            leave_days['Holiday_Date'].notna() &
            (leave_days['Date'] >= leave_days['go_live']) &
            (~leave_days['final_status_clean'].isin(not_eligible_list))
        )
        leave_days['exclusion_weight'] = leave_days['day_weight'].where(leave_days['exclude'], 0.0)
Public Holiday Exemption Logic: Ensures that if an employee's scheduled annual leave overlaps with an officially declared national public holiday, the weight for that day is overridden to 0.0. This ensures the employee's annual leave balance is not deducted for that day, as public holidays are paid rest days by law.

6. Secure Database Uploads (/upload_file)
Python
            for _, row in df.iterrows():
                ...
                stmt = sqlite_upsert(Attendance).values(...)
                update_stmt = stmt.on_conflict_do_update(
                    index_elements=['uae_id', 'Date'],
                    set_={
                        'final_status': stmt.excluded.final_status,
                        'final_status_clean': stmt.excluded.final_status_clean,
                        'go_live_date': stmt.excluded.go_live_date,
                        'queue': stmt.excluded.queue
                    }
                )
                db.execute(update_stmt)
Idempotency Guard: Using SQLite's direct on_conflict_do_update mechanism, the program safely executes an "Upsert". Re-uploading the same attendance or profile spreadsheets simply overrides outdated values without appending duplicate records.

7. High-Performance HTML/JS Interface
JavaScript
// Single Source of Truth UI Binding
const annualEarned = (data.total_earned || 0).toFixed(2);
const annualUsed = (data.total_used_annual || 0).toFixed(2);
const annualRemaining = (data.remaining_annual || 0).toFixed(2);

const casualRemaining = data.casual_remaining !== undefined ? data.casual_remaining : 7;
const sickTaken = data.sick_taken || 0;
const noShowCount = data.noshow_count || 0;

// Injection ensures cards & detailed tabs display identical figures
document.getElementById('val-earned').innerText = annualEarned + ' Days';
document.getElementById('val-used').innerText = annualUsed + ' Days';
document.getElementById('val-remaining').innerText = annualRemaining + ' Days';

document.getElementById('tab-annual-val').innerText = annualRemaining + ' Days';
document.getElementById('tab-casual-val').innerText = casualRemaining + ' Days';
document.getElementById('tab-sick-val').innerText = sickTaken + ' Days';
document.getElementById('tab-noshow-val').innerText = noShowCount + ' Times';
Single Source of Truth: This JavaScript logic binds the API's JSON response directly to both the Top Summary Cards and Detailed Tab Buttons simultaneously. This guarantees that the balances match exactly across all UI elements, preventing discrepancies.

Technical Summary of Features
Precision Calculations: Features fractional calculations for hiring and termination months to ensure precise tenure tracking.

Dynamic Overlap Protections: Automatically detects and exempts public holidays from annual leave deductions.

Optimized Performance: Uses in-memory processing via Pandas DataFrames for quick execution, backed by SQLite and SQLAlchemy.

Data Cleansing: Includes automatic string cleaning (lower() and strip()) on upload to eliminate spelling or formatting mismatches.
