# IDT from fastapi import FastAPI, Request, Form, Depends
from fastapi.responses import HTMLResponse, RedirectResponse
from fastapi.templating import Jinja2Templates
from sqlalchemy.orm import Session
from database import engine, SessionLocal
import models

models.Base.metadata.create_all(bind=engine)

app = FastAPI()
templates = Jinja2Templates(directory="templates")

# ---------------- SESSION STORAGE ----------------
fake_session = {}

# ---------------- DATABASE DEPENDENCY ----------------
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# ---------------- CREATE DEFAULT USERS ----------------
@app.on_event("startup")
def create_default_users():
    db = SessionLocal()
    if not db.query(models.User).first():
        db.add(models.User(username="student1", password="123", role="student"))
        db.add(models.User(username="advisor1", password="123", role="advisor"))
        db.commit()
    db.close()

# ---------------- LOGIN PAGE ----------------
@app.get("/", response_class=HTMLResponse)
def login_page(request: Request):
    return templates.TemplateResponse("login.html", {"request": request})

# ---------------- LOGIN FUNCTION ----------------
@app.post("/login")
def login(request: Request, username: str = Form(...), password: str = Form(...), db: Session = Depends(get_db)):
    user = db.query(models.User).filter_by(username=username, password=password).first()

    if user:
        fake_session["user"] = user.username
        fake_session["role"] = user.role

        if user.role == "student":
            return RedirectResponse("/student", status_code=303)
        else:
            return RedirectResponse("/advisor", status_code=303)

    return {"message": "Invalid Credentials"}

# ---------------- LOGOUT ----------------
@app.get("/logout")
def logout():
    fake_session.clear()
    return RedirectResponse("/", status_code=303)

# ---------------- STUDENT PANEL ----------------
@app.get("/student", response_class=HTMLResponse)
def student_page(request: Request):
    if fake_session.get("role") != "student":
        return RedirectResponse("/", status_code=303)
    return templates.TemplateResponse("student.html", {"request": request})

@app.post("/attendance")
def mark_attendance(name: str = Form(...), db: Session = Depends(get_db)):
    record = models.Attendance(student_name=name, status="Present")
    db.add(record)
    db.commit()
    return {"message": "Attendance Marked"}

@app.post("/noise")
def submit_noise(name: str = Form(...), level: float = Form(...), db: Session = Depends(get_db)):
    record = models.Noise(student_name=name, noise_level=level)
    db.add(record)
    db.commit()
    return {"message": "Noise Submitted"}

@app.post("/location")
def submit_location(name: str = Form(...), location: str = Form(...), db: Session = Depends(get_db)):
    record = models.Location(student_name=name, location=location)
    db.add(record)
    db.commit()
    return {"message": "Location Submitted"}

# ---------------- ADVISOR DASHBOARD ----------------
@app.get("/advisor", response_class=HTMLResponse)
def advisor_dashboard(request: Request, db: Session = Depends(get_db)):
    if fake_session.get("role") != "advisor":
        return RedirectResponse("/", status_code=303)

    attendance = db.query(models.Attendance).all()
    noise = db.query(models.Noise).all()
    location = db.query(models.Location).all()

    return templates.TemplateResponse("advisor.html", {
        "request": request,
        "attendance": attendance,
        "noise": noise,
        "location": location
    })<!DOCTYPE html>
<html>
<head>
    <title>Login</title>
</head>
<body>

<h2>Smart Classroom Login</h2>

<form action="/login" method="post">
    Username: <input type="text" name="username" required><br><br>
    Password: <input type="password" name="password

