# Embu
trade
mpesa-monitor/
├─ backend/
│  ├─ app/
│  │  ├─ main.py
│  │  ├─ config.py
│  │  ├─ models.py
│  │  ├─ schemas.py
│  │  ├─ crud.py
│  │  ├─ db.py
│  │  ├─ tasks.py
│  │  ├─ mpesa_ingest.py
│  │  ├─ notifications.py
│  │  └─ api/
│  │     ├─ transactions.py
│  │     ├─ loans.py
│  │     └─ admin.py
│  ├─ requirements.txt
│  └─ alembic_init.sql
├─ frontend/
│  ├─ package.json
│  └─ src/
│     └─ App.jsx
├─ docker-compose.yml
├─ .env.example
└─ README.md
fastapi==0.95.2
uvicorn[standard]==0.22.0
SQLAlchemy==2.0.20
alembic==1.11.1
psycopg2-binary==2.9.6
pydantic==1.10.12
python-dotenv==1.0.0
celery==5.3.1
redis==4.6.0
httpx==0.24.1
pandas==2.1.3
python-multipart==0.0.6
version: '3.8'
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: mpesa
      POSTGRES_USER: mpesa_user
      POSTGRES_PASSWORD: mpesa_pass
    ports:
      - "5432:5432"
    volumes:
      - db_data:/var/lib/postgresql/data

  redis:
    image: redis:7
    ports:
      - "6379:6379"

  backend:
    build: ./backend
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
    volumes:
      - ./backend:/usr/src/app
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+psycopg2://mpesa_user:mpesa_pass@db:5432/mpesa
      - CELERY_BROKER_URL=redis://redis:6379/0
      - CELERY_RESULT_BACKEND=redis://redis:6379/0
      - ADMIN_EMAILS=you@example.com
    depends_on:
      - db
      - redis

  worker:
    build: ./backend
    command: celery -A app.tasks worker -l info
    volumes:
      - ./backend:/usr/src/app
    environment:
      - DATABASE_URL=postgresql+psycopg2://mpesa_user:mpesa_pass@db:5432/mpesa
      - CELERY_BROKER_URL=redis://redis:6379/0
      - CELERY_RESULT_BACKEND=redis://redis:6379/0
    depends_on:
      - db
      - redis

volumes:
  db_data:
DATABASE_URL=postgresql+psycopg2://mpesa_user:mpesa_pass@localhost:5432/mpesa
CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/0
ADMIN_EMAILS=you@example.com
MPESA_CONSUMER_KEY=
MPESA_CONSUMER_SECRET=
NOTIFICATION_SMS_API_KEY=
-- Run this against your Postgres database to create the basic tables.
CREATE TABLE users (
  id serial PRIMARY KEY,
  name text,
  phone text,
  role text,
  email text,
  password_hash text,
  created_at timestamptz DEFAULT now()
);

CREATE TABLE agents (
  id serial PRIMARY KEY,
  name text,
  mpesa_number text,
  branch text,
  current_cash_balance_estimated numeric DEFAULT 0,
  last_reconciled_at timestamptz
);

CREATE TABLE customers (
  id serial PRIMARY KEY,
  name text,
  phone text,
  national_id text,
  notes text
);

CREATE TABLE loans (
  id serial PRIMARY KEY,
  customer_id integer REFERENCES customers(id),
  agent_id integer REFERENCES agents(id),
  amount numeric NOT NULL,
  purpose text,
  created_at timestamptz DEFAULT now(),
  due_date timestamptz,
  status text DEFAULT 'open',
  reference text
);

CREATE TABLE transactions (
  id serial PRIMARY KEY,
  source_type text,
  source_ref text,
  amount numeric NOT NULL,
  phone text,
  mpesa_code text,
  timestamp timestamptz,
  agent_id integer,
  matched_loan_id integer REFERENCES loans(id),
  status text DEFAULT 'unmatched',
  raw_json jsonb
);

CREATE TABLE commissions (
  id serial PRIMARY KEY,
  agent_id integer REFERENCES agents(id),
  transaction_id integer REFERENCES transactions(id),
  amount numeric,
  recorded_at timestamptz DEFAULT now()
);

CREATE TABLE alerts (
  id serial PRIMARY KEY,
  loan_id integer REFERENCES loans(id),
  alert_type text,
  sent_to text,
  channel text,
  sent_at timestamptz DEFAULT now(),
  status text
);

CREATE TABLE daily_summaries (
  id serial PRIMARY KEY,
  day_date date,
  total_disbursed numeric,
  total_repaid numeric,
  total_commission numeric,
  agent_positions jsonb,
  created_at timestamptz DEFAULT now()
);
import os
from dotenv import load_dotenv
load_dotenv()

DATABASE_URL = os.getenv("DATABASE_URL")
CELERY_BROKER_URL = os.getenv("CELERY_BROKER_URL")
CELERY_RESULT_BACKEND = os.getenv("CELERY_RESULT_BACKEND")
ADMIN_EMAILS = os.getenv("ADMIN_EMAILS", "")
MPESA_CONSUMER_KEY = os.getenv("MPESA_CONSUMER_KEY", "")
MPESA_CONSUMER_SECRET = os.getenv("MPESA_CONSUMER_SECRET", "")
NOTIFICATION_SMS_API_KEY = os.getenv("NOTIFICATION_SMS_API_KEY", "")
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base
from .config import DATABASE_URL

engine = create_engine(DATABASE_URL, future=True)
SessionLocal = sessionmaker(bind=engine, autocommit=False, autoflush=False, future=True)
Base = declarative_base()

# Dependency
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
from sqlalchemy import Column, Integer, String, Numeric, DateTime, Text, JSON, ForeignKey, Date
from sqlalchemy.sql import func
from .db import Base
from sqlalchemy.orm import relationship

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(Text)
    phone = Column(Text)
    role = Column(Text)
    email = Column(Text)
    password_hash = Column(Text)
    created_at = Column(DateTime(timezone=True), server_default=func.now())

class Agent(Base):
    __tablename__ = "agents"
    id = Column(Integer, primary_key=True)
    name = Column(Text)
    mpesa_number = Column(Text)
    branch = Column(Text)
    current_cash_balance_estimated = Column(Numeric, default=0)
    last_reconciled_at = Column(DateTime(timezone=True))

class Customer(Base):
    __tablename__ = "customers"
    id = Column(Integer, primary_key=True)
    name = Column(Text)
    phone = Column(Text)
    national_id = Column(Text)
    notes = Column(Text)

class Loan(Base):
    __tablename__ = "loans"
    id = Column(Integer, primary_key=True)
    customer_id = Column(Integer, ForeignKey("customers.id"))
    agent_id = Column(Integer, ForeignKey("agents.id"))
    amount = Column(Numeric, nullable=False)
    purpose = Column(Text)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    due_date = Column(DateTime(timezone=True))
    status = Column(Text, default="open")
    reference = Column(Text)

    customer = relationship("Customer")
    agent = relationship("Agent")
    transactions = relationship("Transaction", back_populates="loan")

class Transaction(Base):
    __tablename__ = "transactions"
    id = Column(Integer, primary_key=True)
    source_type = Column(Text)
    source_ref = Column(Text)
    amount = Column(Numeric, nullable=False)
    phone = Column(Text)
    mpesa_code = Column(Text)
    timestamp = Column(DateTime(timezone=True))
    agent_id = Column(Integer)
    matched_loan_id = Column(Integer, ForeignKey("loans.id"))
    status = Column(Text, default="unmatched")
    raw_json = Column(JSON)

    loan = relationship("Loan", back_populates="transactions", foreign_keys=[matched_loan_id])

class Commission(Base):
    __tablename__ = "commissions"
    id = Column(Integer, primary_key=True)
    agent_id = Column(Integer, ForeignKey("agents.id"))
    transaction_id = Column(Integer, ForeignKey("transactions.id"))
    amount = Column(Numeric)
    recorded_at = Column(DateTime(timezone=True), server_default=func.now())

class Alert(Base):
    __tablename__ = "alerts"
    id = Column(Integer, primary_key=True)
    loan_id = Column(Integer, ForeignKey("loans.id"))
    alert_type = Column(Text)
    sent_to = Column(Text)
    channel = Column(Text)
    sent_at = Column(DateTime(timezone=True), server_default=func.now())
    status = Column(Text)

class DailySummary(Base):
    __tablename__ = "daily_summaries"
    id = Column(Integer, primary_key=True)
    day_date = Column(Date)
    total_disbursed = Column(Numeric)
    total_repaid = Column(Numeric)
    total_commission = Column(Numeric)
    agent_positions = Column(JSON)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
from pydantic import BaseModel
from typing import Optional, Any
from datetime import datetime, date

class TransactionCreate(BaseModel):
    source_type: str
    source_ref: Optional[str]
    amount: float
    phone: Optional[str]
    mpesa_code: Optional[str]
    timestamp: Optional[datetime]
    agent_id: Optional[int]
    raw_json: Optional[Any]

class TransactionOut(TransactionCreate):
    id: int
    status: str

    class Config:
        orm_mode = True

class LoanCreate(BaseModel):
    customer_id: int
    agent_id: Optional[int]
    amount: float
    purpose: Optional[str]
    due_date: Optional[datetime]
    reference: Optional[str]

class LoanOut(LoanCreate):
    id: int
    status: str
    created_at: datetime

    class Config:
        orm_mode = True
from sqlalchemy.orm import Session
from . import models, schemas
from sqlalchemy import select, update, func
from datetime import datetime, timedelta

def create_transaction(db: Session, tx: schemas.TransactionCreate):
    t = models.Transaction(
        source_type=tx.source_type,
        source_ref=tx.source_ref,
        amount=tx.amount,
        phone=tx.phone,
        mpesa_code=tx.mpesa_code,
        timestamp=tx.timestamp,
        agent_id=tx.agent_id,
        raw_json=tx.raw_json,
        status="unmatched"
    )
    db.add(t)
    db.commit()
    db.refresh(t)
    return t

def get_unmatched_transactions(db: Session):
    return db.query(models.Transaction).filter(models.Transaction.status == "unmatched").all()

def create_loan(db: Session, loan: schemas.LoanCreate):
    l = models.Loan(
        customer_id=loan.customer_id,
        agent_id=loan.agent_id,
        amount=loan.amount,
        purpose=loan.purpose,
        due_date=loan.due_date,
        reference=loan.reference,
        status="open"
    )
    db.add(l)
    db.commit()
    db.refresh(l)
    return l

def get_open_loans(db: Session):
    return db.query(models.Loan).filter(models.Loan.status == "open").all()

def match_transaction_to_loan(db: Session, tx: models.Transaction, loan: models.Loan):
    tx.matched_loan_id = loan.id
    tx.status = "matched"
    loan.status = "paid" if tx.amount >= loan.amount else "partial"
    db.commit()
    db.refresh(tx)
    db.refresh(loan)
    return tx, loan

def find_candidate_loans(db: Session, amount: float, phone: str = None, time_window_hours: int = 48):
    q = db.query(models.Loan).filter(models.Loan.status == "open")
    q = q.filter(models.Loan.amount <= amount * 1.02)  # allow small differences
    if phone:
        customer_ids = db.query(models.Customer.id).filter(models.Customer.phone == phone).subquery()
        q = q.filter(models.Loan.customer_id.in_(customer_ids))
    cutoff = datetime.utcnow() - timedelta(hours=time_window_hours)
    q = q.filter(models.Loan.created_at >= cutoff)
    return q.order_by(models.Loan.created_at.asc()).all()

def record_commission(db: Session, agent_id: int, transaction_id: int, amount: float):
    c = models.Commission(agent_id=agent_id, transaction_id=transaction_id, amount=amount)
    db.add(c)
    db.commit()
    db.refresh(c)
    return c
# simple notification stubs (replace with real provider integration)
from .config import ADMIN_EMAILS, NOTIFICATION_SMS_API_KEY
import httpx
import os

def send_sms(phone: str, message: str):
    # Replace this stub with Africa's Talking / Twilio call
    print(f"[SMS -> {phone}] {message}")
    # Example with httpx for a provider:
    # r = httpx.post("https://sms-provider/send", json={"to": phone, "message": message}, headers={...})

def send_email(to: str, subject: str, body: str):
    # Replace with SendGrid / SMTP integration
    print(f"[EMAIL -> {to}] {subject}\n{body}")

def notify_admins(subject: str, body: str):
    for e in ADMIN_EMAILS.split(","):
        if e.strip():
            send_email(e.strip(), subject, body)
import csv
from io import StringIO
from datetime import datetime
from .schemas import TransactionCreate
from .crud import create_transaction

def parse_mpesa_csv_and_create(db, csv_content: str):
    """
    Expect CSV with columns (timestamp, amount, partyA, partyB, mpesa_code, narrative)
    Adapt parsing as needed to match your agent statement format.
    """
    reader = csv.DictReader(StringIO(csv_content))
    created = []
    for row in reader:
        ts = None
        for key in ("timestamp", "time", "date"):
            if key in row and row[key]:
                try:
                    ts = datetime.fromisoformat(row[key])
                    break
                except Exception:
                    try:
                        ts = datetime.strptime(row[key], "%Y-%m-%d %H:%M:%S")
                        break
                    except Exception:
                        ts = None
        tx_in = TransactionCreate(
            source_type="MPESA_AGENT",
            source_ref=row.get("narrative") or row.get("transaction"),
            amount=float(row.get("amount") or row.get("Amount") or 0),
            phone=row.get("partyA") or row.get("phone"),
            mpesa_code=row.get("mpesa_code") or row.get("transaction_id"),
            timestamp=ts,
            agent_id=None,
            raw_json=row
        )
        t = create_transaction(db, tx_in)
        created.append(t)
    return created
from celery import Celery
from .config import CELERY_BROKER_URL, CELERY_RESULT_BACKEND
from .db import SessionLocal
from . import crud, notifications, models
from datetime import datetime, timedelta
import os

celery = Celery("tasks", broker=CELERY_BROKER_URL, backend=CELERY_RESULT_BACKEND)

@celery.on_after_configure.connect
def setup_periodic_tasks(sender, **kwargs):
    # run reconcile every day at 00:05 (cron schedule) - adjust per deployment
    sender.add_periodic_task(60.0 * 60 * 24, reconcile_and_send_daily_summary.s(), name='daily_reconcile')

@celery.task
def reconcile_and_send_daily_summary():
    db = SessionLocal()
    try:
        # 1. Attempt to auto-match unmatched transactions
        unmatched = crud.get_unmatched_transactions(db)
        for tx in unmatched:
            candidates = crud.find_candidate_loans(db, float(tx.amount), phone=tx.phone)
            if candidates:
                # naive: match to earliest candidate
                loan = candidates[0]
                crud.match_transaction_to_loan(db, tx, loan)
                # optional: record commission for agent if present (example 1%),
                if tx.agent_id:
                    fee = float(tx.amount) * 0.01
                    crud.record_commission(db, tx.agent_id, tx.id, fee)
        # 2. Create daily summary
        today = datetime.utcnow().date()
        total_disbursed = db.query(models.Loan).filter(models.Loan.created_at >= datetime.utcnow() - timedelta(days=1)).with_entities(func.coalesce(func.sum(models.Loan.amount),0)).scalar()
        total_repaid = db.query(models.Transaction).filter(models.Transaction.timestamp >= datetime.utcnow() - timedelta(days=1), models.Transaction.status == "matched").with_entities(func.coalesce(func.sum(models.Transaction.amount),0)).scalar()
        total_commission = db.query(models.Commission).filter(models.Commission.recorded_at >= datetime.utcnow() - timedelta(days=1)).with_entities(func.coalesce(func.sum(models.Commission.amount),0)).scalar()
        # agent positions - simple example
        agents = db.query(models.Agent).all()
        agent_positions = {a.id: float(a.current_cash_balance_estimated or 0) for a in agents}
        summary = models.DailySummary(
            day_date=today,
            total_disbursed=total_disbursed or 0,
            total_repaid=total_repaid or 0,
            total_commission=total_commission or 0,
            agent_positions=agent_positions
        )
        db.add(summary)
        db.commit()
        # 3. Overdue alerts
        overdue_cutoff = datetime.utcnow() - timedelta(days=1) # configurable grace 1 day
        overdue_loans = db.query(models.Loan).filter(models.Loan.due_date != None, models.Loan.due_date < datetime.utcnow(), models.Loan.status == "open").all()
        for loan in overdue_loans:
            # send reminder
            if loan.customer and loan.customer.phone:
                notifications.send_sms(loan.customer.phone, f"Reminder: Loan {loan.id} of KES {loan.amount} is overdue.")
                # record alert
                a = models.Alert(loan_id=loan.id, alert_type='overdue', sent_to=loan.customer.phone, channel='sms', status='sent')
                db.add(a)
        db.commit()
        # notify admins
        notifications.notify_admins("Daily summary ready", f"Disbursed: {total_disbursed}, Repaid: {total_repaid}, Commission: {total_commission}")
    except Exception as e:
        print("Error in reconcile task", e)
    finally:
        db.close()
from fastapi import APIRouter, Depends, UploadFile, File, HTTPException
from ..db import get_db
from sqlalchemy.orm import Session
from .. import crud, schemas, mpesa_ingest
import io

router = APIRouter(prefix="/transactions", tags=["transactions"])

@router.post("/upload-csv")
async def upload_csv(file: UploadFile = File(...), db: Session = Depends(get_db)):
    content = await file.read()
    try:
        created = mpesa_ingest.parse_mpesa_csv_and_create(db, content.decode())
        return {"created": len(created)}
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.post("/")
def create_transaction(tx: schemas.TransactionCreate, db: Session = Depends(get_db)):
    return crud.create_transaction(db, tx)

@router.get("/unmatched")
def get_unmatched(db: Session = Depends(get_db)):
    return crud.get_unmatched_transactions(db)
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from .. import schemas, crud
from ..db import get_db

router = APIRouter(prefix="/loans", tags=["loans"])

@router.post("/")
def create_loan(loan: schemas.LoanCreate, db: Session = Depends(get_db)):
    return crud.create_loan(db, loan)

@router.get("/open")
def open_loans(db: Session = Depends(get_db)):
    return crud.get_open_loans(db)
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session
from ..db import get_db
from .. import models

router = APIRouter(prefix="/admin", tags=["admin"])

@router.get("/daily-summary")
def daily_summaries(db: Session = Depends(get_db)):
    return db.query(models.DailySummary).order_by(models.DailySummary.day_date.desc()).limit(10).all()
from fastapi import FastAPI
from .db import engine
from . import models
from .api import transactions, loans, admin
from .config import DATABASE_URL

app = FastAPI(title="MPESA Agent Loan Monitoring")

# Create tables if not present (for MVP)
models.Base.metadata.create_all(bind=engine)

app.include_router(transactions.router)
app.include_router(loans.router)
app.include_router(admin.router)

@app.get("/")
def root():
    return {"status": "ok"}
FROM python:3.11-slim

WORKDIR /usr/src/app
COPY ./requirements.txt /usr/src/app/requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

COPY . /usr/src/app

ENV PYTHONPATH=/usr/src/app
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
{
  "name": "mpesa-monitor-frontend",
  "version": "0.1.0",
  "private": true,
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "vite": "^5.0.0"
  },
  "scripts": {
    "dev": "vite"
  }
}
import React, { useEffect, useState } from "react";

export default function App() {
  const [unmatched, setUnmatched] = useState([]);
  const [openLoans, setOpenLoans] = useState([]);
  const [summary, setSummary] = useState([]);

  useEffect(() => {
    fetch("/transactions/unmatched").then(r => r.json()).then(setUnmatched).catch(()=>{});
    fetch("/loans/open").then(r => r.json()).then(setOpenLoans).catch(()=>{});
    fetch("/admin/daily-summary").then(r => r.json()).then(setSummary).catch(()=>{});
  }, []);

  return (
    <div style={{padding:20, fontFamily: 'Arial'}}>
      <h1>MPESA Monitor — Dashboard</h1>
      <section style={{marginTop:20}}>
        <h2>Unmatched Transactions</h2>
        <table border="1" cellPadding="8">
          <thead><tr><th>ID</th><th>Amount</th><th>Phone</th><th>MPESA Code</th><th>Timestamp</th></tr></thead>
          <tbody>
            {unmatched.map(u => (
              <tr key={u.id}>
                <td>{u.id}</td>
                <td>{u.amount}</td>
                <td>{u.phone}</td>
                <td>{u.mpesa_code}</td>
                <td>{u.timestamp}</td>
              </tr>
            ))}
          </tbody>
        </table>
      </section>

      <section style={{marginTop:20}}>
        <h2>Open Loans</h2>
        <table border="1" cellPadding="8">
          <thead><tr><th>ID</th><th>Customer</th><th>Amount</th><th>Due</th></tr></thead>
          <tbody>
            {openLoans.map(l => (
              <tr key={l.id}>
                <td>{l.id}</td>
                <td>{l.customer_id}</td>
                <td>{l.amount}</td>
                <td>{l.due_date}</td>
              </tr>
            ))}
          </tbody>
        </table>
      </section>

      <section style={{marginTop:20}}>
        <h2>Daily Summaries (recent)</h2>
        <ul>
          {summary.map(s => (
            <li key={s.id}>{s.day_date} — Disbursed: {s.total_disbursed} Repaid: {s.total_repaid} Commission: {s.total_commission}</li>
          ))}
        </ul>
      </section>
    </div>
  );
}
# MPESA Agent Loan Monitoring — MVP

## Overview
This repo provides an MVP to monitor MPESA agent transactions used to disburse loans and reconcile repayments. It includes:
- FastAPI backend with endpoints to ingest transactions and manage loans
- Celery worker for daily reconciliation/alerts
- Simple React dashboard
- Docker Compose for local dev (Postgres + Redis + Backend + Worker)

## Quickstart (local)
1. Copy `.env.example` to `.env` and tweak values.
2. From repo root:
   ```bash
   docker compose up --build
