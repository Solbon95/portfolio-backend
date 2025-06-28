# Портфолио-бэкенд

Добро пожаловать в репозиторий backend-проекта портфолио начинающего разработчика. Здесь размещён код, демонстрирующий мои навыки работы с серверной частью приложений.

## 🛠 Стек технологий

- **FastAPI** — современный Python-фреймворк для создания API
- **PostgreSQL** — надёжная реляционная база данных
- **SQLAlchemy** — ORM для работы с БД через Python-код
- **Pydantic** — валидация и сериализация данных
- **Alembic** — миграции схемы базы данных
- **Docker & Docker Compose** — контейнеризация и управление сервисами
- **python-dotenv** — хранение конфиденциальных настроек
- **Uvicorn** — ASGI-сервер для запуска FastAPI

## 🚀 Возможности API

- Создание проектов
- Получение списка проектов

## 📁 Структура проекта

```text
portfolio-backend/
├── app/
│   ├── main.py           # Точка входа
│   ├── database.py       # Настройка подключения к БД 
│   ├── models/           # SQLAlchemy-модели
│   │   └── project.py
│   ├── schemas/          # Pydantic-схемы
│   │   └── project.py
│   ├── api/              # Роуты FastAPI
│   │   ├── __init__.py   # __init__.py
│   │   └── projects.py
├── .env                  # Загружаем переменные окружения DATABASE_URL=postgresql://postgres:password@localhost:5432/portfolio_db
├── Dockerfile            # Docker-инструкция 
├── docker-compose.yml    # Конфигурация сервисов
├── requirements.txt      # Зависимости Python
└── README.md             # Документация проекта
```

```bash
git clone https://github.com/your-username/portfolio-backend.git
cd portfolio-backend
```

```env
DATABASE_URL=postgresql://postgres:password@localhost:5432/portfolio_db
```



from sqlalchemy import create_engine
from app.db.base import Base
from sqlalchemy.orm import sessionmaker
import os
from dotenv import load_dotenv
from app.db.base import Base  # Добавь если используешь Base отдельно



load_dotenv() # загружаем переменные из .env


DATABASE_URL = os.getenv("DATABASE_URL") # читаем переменную из окружения

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)




from sqlalchemy import Column, Integer, String, Text
from app.db.base import Base # Импортируешь базовый класс для моделей


class Project(Base):
    __tablename__ = "projects"

    id = Column(Integer, primary_key=True, index=True)    
    title = Column(String(255), nullable=False, unique=True, index=True)    
    description = Column(Text, nullable=True)




from pydantic import BaseModel
class ProjectBase(BaseModel):
    
    title: str
    
    description: str | None = None

class ProjectCreate(ProjectBase):
    
    pass

class ProjectRead(ProjectBase):
    id: int

    class Config:
        orm_mode = True




from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List

from app import models, schemas
from app.database import SessionLocal

router = APIRouter(prefix="/projects", tags=["projects"])

def get_db():
      
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@router.post("/", response_model=schemas.ProjectRead)
def create_project(project: schemas.ProjectCreate, db: Session = Depends(get_db)):
      
    db_project = db.query(models.Project).filter(models.Project.title == project.title).first()
    if db_project:
        raise HTTPException(status_code=400, detail="Project with this title already exists")
    new_project = models.Project(title=project.title, description=project.description)
    db.add(new_project)
    db.commit()
    db.refresh(new_project)
    return new_project

@router.get("/", response_model=List[schemas.ProjectRead])
def read_projects(skip: int = 0, limit: int = 10, db: Session = Depends(get_db)):
    
    projects = db.query(models.Project).offset(skip).limit(limit).all()
    return projects




from fastapi import FastAPI
from app.api import projects
from app.db.base import Base
from app.database import engine

app = FastAPI(
    title="Portfolio API",
    version="1.0.0",
    description="Проектное API с FastAPI и SQLAlchemy"
)

# Создание таблиц при запуске (dev only, для прода использовать Alembic)
Base.metadata.create_all(bind=engine)

# Подключаем роуты
app.include_router(projects.router)




