from fastapi import FastAPI, Request, Form
from fastapi.responses import HTMLResponse, JSONResponse
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates
import sqlite3
from datetime import datetime

app = FastAPI(title="LegalEase - AI Powered Document Generator")

app.mount("/static", StaticFiles(directory="static"), name="static")
templates = Jinja2Templates(directory="templates")

DB_NAME = "legalease.db"

def init_db():
    with sqlite3.connect(DB_NAME) as conn:
        conn.execute(""" CREATE TABLE IF NOT EXISTS documents ( id INTEGER PRIMARY KEY AUTOINCREMENT, document_type TEXT NOT NULL, title TEXT NOT NULL, content TEXT NOT NULL, created_at TEXT NOT NULL ) """)

def generate_document(document_type, data):
    if document_type == "Rental Agreement":
        return f"""RENTAL AGREEMENT This Rental Agreement is made between {data.get('party1', 'Landlord')} and {data.get('party2', 'Tenant')}. Property: {data.get('property', 'Not specified')} Monthly Rent: {data.get('amount', 'Not specified')} Security Deposit: {data.get('deposit', 'Not specified')} Lease Duration: {data.get('duration', 'Not specified')} TERMS 1. The tenant agrees to pay the monthly rent on the agreed payment date. 2. The tenant shall maintain the property in a reasonable condition. 3. The security deposit will be handled according to the applicable agreement and law. 4. Any additional terms should be reviewed and agreed by the parties. DISCLAIMER This is a demonstration draft and should be reviewed by a qualified legal professional before use."""

    if document_type == "Employment Agreement":
        return f"""EMPLOYMENT AGREEMENT Employer: {data.get('party1', 'Employer')} Employee: {data.get('party2', 'Employee')} Designation: {data.get('designation', 'Not specified')} Salary: {data.get('amount', 'Not specified')} Joining Date: {data.get('date', 'Not specified')} TERMS 1. The employee will perform duties associated with the stated designation. 2. Compensation will be handled according to the agreed employment terms. 3. Working conditions, leave, confidentiality and termination provisions should be finalized by the parties. 4. Applicable employment laws and organizational policies should be considered. DISCLAIMER This is a demonstration draft and should be reviewed by a qualified legal professional before use."""

    if document_type == "Non-Disclosure Agreement":
        return f"""NON-DISCLOSURE AGREEMENT Disclosing Party: {data.get('party1', 'Disclosing Party')} Receiving Party: {data.get('party2', 'Receiving Party')} Purpose: {data.get('purpose', 'Not specified')} Confidentiality Period: {data.get('duration', 'Not specified')} TERMS 1. Confidential information shall be used only for the stated purpose. 2. The receiving party should take reasonable steps to protect confidential information. 3. Information that is publicly available or independently obtained may be subject to agreed exclusions. 4. The parties should finalize the applicable obligations and remedies. DISCLAIMER This is a demonstration draft and should be reviewed by a qualified legal professional before use."""

    return f"""LEGAL NOTICE / LETTER To: {data.get('party2', 'Recipient')} From: {data.get('party1', 'Sender')} Date: {data.get('date', datetime.now().strftime('%Y-%m-%d'))} SUBJECT: {data.get('subject', 'Legal Notice')} BACKGROUND {data.get('issue', 'Please provide the relevant facts and issue description.')} REQUESTED ACTION {data.get('action', 'Please provide the requested action or response.')} CLOSING This document is a structured demonstration draft based on the information provided by the user. DISCLAIMER This is a demonstration draft and should be reviewed by a qualified legal professional before use."""

@app.on_event("startup")
def startup():
    init_db()

@app.get("/", response_class=HTMLResponse)
def home(request: Request):
    return templates.TemplateResponse("index.html", {"request": request})

@app.post("/generate-document")
def generate( document_type: str = Form(...), party1: str = Form(""), party2: str = Form(""), property: str = Form(""), amount: str = Form(""), deposit: str = Form(""), duration: str = Form(""), designation: str = Form(""), date: str = Form(""), purpose: str = Form(""), subject: str = Form(""), issue: str = Form(""), action: str = Form("") ):
    data = locals()
    content = generate_document(document_type, data)
    title = f"{document_type} - {party1 or 'Draft'}"

    with sqlite3.connect(DB_NAME) as conn:
        cur = conn.execute(
            "INSERT INTO documents (document_type,title,content,created_at) VALUES (?,?,?,?)",
            (document_type, title, content, datetime.now().isoformat(timespec="seconds"))
        )
        doc_id = cur.lastrowid

    return JSONResponse({"id": doc_id, "title": title, "content": content})

@app.get("/history")
def history():
    with sqlite3.connect(DB_NAME) as conn:
        rows = conn.execute(
            "SELECT id, document_type, title, created_at FROM documents ORDER BY id DESC"
        ).fetchall()
    return [{"id": r[0], "document_type": r[1], "title": r[2], "created_at": r[3]} for r in rows]

@app.get("/document/{doc_id}")
def get_document(doc_id: int):
    with sqlite3.connect(DB_NAME) as conn:
        row = conn.execute(
            "SELECT id, document_type, title, content, created_at FROM documents WHERE id=?",
            (doc_id,)
        ).fetchone()
    if not row:
        return JSONResponse({"error": "Document not found"}, status_code=404)
    return {
        "id": row[0], "document_type": row[1], "title": row[2],
        "content": row[3], "created_at": row[4]
    }
