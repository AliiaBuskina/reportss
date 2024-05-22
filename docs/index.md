# Добро пожаловать в ToDone API Documentation!

Этот сайт предоставляет документацию для ToDone API - API для менеджера задач.

## Обзор

ToDone API предоставляет возможность управления пользователями и задачами. Вы можете создавать, обновлять, удалять и просматривать пользователей и их задачи, используя соответствующие конечные точки API.
# main.py
```python
from fastapi import FastAPI
from connection import *
from routes.task_routes import router as task_router
from routes.user_routes import router as user_router


app = FastAPI(
    title="ToDone",
    description="API для менеджера задач",
    version="1.0.0"
)

@app.on_event("startup")
def on_startup():
    print("Initializing database...")
    init_db()
    print("Database initialized.")

user_tags_metadata = {"Operations related to users"}
task_tags_metadata = {"Operations related to tasks"}

app.include_router(user_router, tags=[user_tags_metadata])
app.include_router(task_router, tags=[task_tags_metadata])

```
## Доступные разделы

- [API пользователей](user_endpoints.md): Описание конечных точек, связанных с пользователями.
- [API задач](task_endpoints.md): Описание конечных точек, связанных с задачами.
- [Models](models.md): модель данных
- [Schemas](schemas.md): Pydantic-схемы


## (пока в разработке)
## Получение доступа к API

Для доступа к API вам необходимо быть авторизованным пользователем. Пожалуйста, получите доступ к токену аутентификации, используя конечную точку `/token`, а затем включите этот токен в заголовок `Authorization` при отправке запросов к API.

## Примеры использования

### Получение списка всех пользователей

