modern_fastapi_app/
│
├── app/
│   ├── main.py
│   │       from fastapi import FastAPI
│   │       from fastapi.staticfiles import StaticFiles
│   │       from app.routes import router as app_router
│   │       from app.database import Base, engine
            
│   │
│   │       Base.metadata.create_all(bind=engine)
│   │
│   │       app = FastAPI(title="Modern FastAPI App", version="1.0.0")
│   │       app.include_router(app_router)
│   │
│   │       app.mount("/static", StaticFiles(directory="static"), name="static")
│   │
│   ├── database.pys
│   │       from sqlalchemy import create_
│   │       from sqlalchemy.ext.declarative import declarative_base
│   │       from sqlalchemy.orm import sessionmaker
│   │
│   │       SQLALCHEMY_DATABASE_URL = "sqlite:///./app.db"
│   │
│   │       engine = create_engine(SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False})
│   │       SessionLocal = sessionmaker(bind=engine, autocommit=False, autoflush=False)
│   │       Base = declarative_base()
│   │
│   │       def get_db():
│   │           db = SessionLocal()
│   │           try:
│   │               yield db
│   │           finally:
│   │               db.close()
│   │
│   ├── models.pys
│   │       from sqlalchemy import Column, Integer, String, Text
│   │       from app.database import Base
│   │
│   │       class User(Base):
│   │           __tablename__ = "users"
│   │           id = Column(Integer, primary_key=True, index=True)
│   │           username = Column(String(150), unique=True, nullable=False)
│   │           password = Column(String(255), nullable=False)
│   │
│   │       class Message(Base):
│   │           __tablename__ = "messages"
│   │           id = Column(Integer, primary_key=True, index=True)
│   │           name = Column(String(150), nullable=False)
│   │           email = Column(String(150), nullable=False)
│   │           message = Column(Text, nullable=False)
│   │
│   ├── auth.pys
│   │       
│   │       from datetime import datetime, timedelta
│   │       from fastapi import Request
│   │       from sqlalchemy.orm import Session
│   │       from werkzeug.security import generate_password_hash, check_password_hash
│   │       from app.database import get_db
│   │       from app.models import User
│   │
│   │       SECRET_KEY = "supersecretkey"
│   │       ALGORITHM = "HS256"
│   │
│   │       def create_user(db: Session, username: str, password: str):
│   │           hashed_pw = generate_password_hash(password)
│   │           user = User(username=username, password=hashed_pw)
│   │           db.add(user)
│   │           db.commit()
│   │
│   │       def authenticate_user(db: Session, username: str, password: str):
│   │           user = db.query(User).filter(User.username == username).first()
│   │           if user and check_password_hash(user.password, password):
│   │               token_data = {"sub": username, "exp": datetime.utcnow() + timedelta(hours=1)}
│   │               return jwt.encode(token_data, SECRET_KEY, algorithm=ALGORITHM)
│   │           return None
│   │
│   │       def get_current_user(request: Request):
│   │           token = request.cookies.get("token")
│   │           if not token:
│   │               return None
│   │           try:.py
│   │               payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
│   │               return payload.get("sub")
│   │           except jwt.ExpiredSignatureError:
│   │               return None
│   │           except jwt.InvalidTokenError:
│   │               return None
│   │
│   ├── routes.py
│   │       from fastapi import APIRouter, Request, Form, Depends
│   │       from fastapi.responses import HTMLResponse, RedirectResponse
│   │       from fastapi.templating import Jinja2Templates
│   │       from sqlalchemy.orm import Session
│   │       from app.database import get_db
│   │       from app.models import Message
│   │       from app.auth import create_user, authenticate_user, get_current_user
│   │
│   │       router = APIRouter()
│   │       templates = Jinja2Templates(directory="app/templates")
│   │
│   │       @router.get("/", response_class=HTMLResponse)
│   │       def home(request: Request, user: str = Depends(get_current_user)):
│   │           return templates.TemplateResponse("index.html", {"request": request, "user": user})
│   │
│   │       @router.get("/register", response_class=HTMLResponse)
│   │       def register_form(request: Request):
│   │           return templates.TemplateResponse("register.html", {"request": request})
│   │
│   │       @router.post("/register")
│   │       def register(username: str = Form(...), password: str = Form(...), db: Session = Depends(get_db)):
│   │           create_user(db, username, password)
│   │           return RedirectResponse(url="/login", status_code=303)
│   │
│   │       @router.get("/login", response_class=HTMLResponse)
│   │       def login_form(request: Request):
│   │           return templates.TemplateResponse("login.html", {"request": request})
│   │
│   │       @router.post("/login")
│   │       def login(username: str = Form(...), password: str = Form(...), db: Session = Depends(get_db)):
│   │           token = authenticate_user(db, username, password)
│   │           if token:
│   │               response = RedirectResponse(url="/", status_code=303)
│   │               response.set_cookie("token", token, httponly=True)
│   │               return response
│   │           return RedirectResponse(url="/login", status_code=303)
│   │
│   │       @router.get("/logout")
│   │       def logout():
│   │           response = RedirectResponse(url="/")
│   │           response.delete_cookie("token")
│   │           return response
│   │
│   │       @router.get("/contact", response_class=HTMLResponse)
│   │       def contact_form(request: Request, user: str = Depends(get_current_user)):
│   │           return templates.TemplateResponse("contact.html", {"request": request, "user": user})
│   │
│   │       @router.post("/contact")
│   │       def contact(name: str = Form(...), email: str = Form(...), message: str = Form(...), db: Session = Depends(get_db)):
│   │           msg = Message(name=name, email=email, message=message)
│   │           db.add(msg)
│   │           db.commit()
│   │           return RedirectResponse(url="/contact", status_code=303)
│   │
│   │       @router.get("/admin", response_class=HTMLResponse)
│   │       def admin_dashboard(request: Request, db: Session = Depends(get_db), user: str = Depends(get_current_user)):
│   │           if not user:
│   │               return RedirectResponse(url="/login", status_code=303)
│   │           messages = db.query(Message).all()
│   │           return templates.TemplateResponse("admin.html", {"request": request, "user": user, "messages": messages})
│   │
│   └── templates/
│       ├── 
│       │       <!DOCTYPE html>
│       │       <html lang="en">
│       │       <head>
│       │         <meta charset="UTF-8">
│       │         <meta name="viewport" content="width=device-width, initial-scale=1.0">
│       │         <title>{{ user or "FastAPI App" }}</title>
│       │         <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
│       │       </head>
│       │       <body>
│       │       <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
│       │         <div class="container">
│       │           <a class="navbar-brand" href="/">FastAPI App</a>
│       │           <div>
│       │             <a class="nav-link d-inline text-white" href="/">Home</a>
│       │             <a class="nav-link d-inline text-white" href="/contact">Contact</a>
│       │             {% if user %}
│       │               <a class="nav-link d-inline text-warning" href="/admin">Admin</a>
│       │               <a class="nav-link d-inline text-danger" href="/logout">Logout</a>
│       │             {% else %}
│       │               <a class="nav-link d-inline text-info" href="/login">Login</a>
│       │               <a class="nav-link d-inline text-success" href="/register">Register</a>
│       │             {% endif %}
│       │           </div>
│       │         </div>
│       │       </nav>
│       │       <div class="container mt-4">
│       │         {% block content %}{% endblock %}
│       │       </div>
│       │       </body>
│       │       </html>
│       │
│       ├── index.html
│       ├── login.html
│       ├── register.html
│       ├── contact.html
│       └── admin.html
│
├── requirements.txt
│       fastapi
│       uvicorn
│       sqlalchemy
│       jinja2
│       werkzeug
│       pyjwt
│
└── Dockerfile
        FROM python:3.11-slim
        WORKDIR /app
        COPY . /app
        RUN pip install --no-cache-dir -r requirements.txt
        CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]

