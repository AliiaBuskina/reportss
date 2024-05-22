# Эндпоинты для пользователя

```python
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.exceptions import RequestValidationError
import jwt
from sqlalchemy.orm import Session
import models, crud, schemas
from auth import auth
from connection import get_session
from schemas import *
import auth.auth

router = APIRouter()


@router.post("/register", response_model=schemas.UserCreate)
def register_user(user: schemas.UserCreate, db: Session = Depends(get_session)):
    db_user = crud.get_user_by_username(db, username=user.username)
    if db_user:
        raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail="Username already registered")
    return auth.create_user(db=db, user=user)


@router.post("/login")
def login_user(user: schemas.UserLogin, db: Session = Depends(get_session)):
    try:
        authenticated_user = auth.authenticate_user(db=db, username=user.username, password=user.password)
        if not authenticated_user:
            raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid username or password")
        access_token = auth.create_access_token(data={"sub": authenticated_user.username})
        return {"access_token": access_token, "token_type": "bearer"}
    except RequestValidationError as e:
        raise HTTPException(status_code=422, detail="Validation error", headers=e.headers)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


async def get_current_user(db: Session = Depends(get_session), token: str = Depends(auth.oauth2_scheme)):
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, auth.SECRET_KEY, algorithms=[auth.ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
    except jwt.JWTError:
        raise credentials_exception
    user = crud.get_user_by_username(db, username)
    if user is None:
        raise credentials_exception
    return user


@router.post("/users/",
             response_model=UserRead,
             status_code=status.HTTP_201_CREATED,
             tags=["Users"],  # Теги для организации в документации
             summary="Create a new user",  # Краткое описание метода
             description="Create a new user with the provided data.",
             responses={  # Примеры ответов
                 201: {"description": "User created successfully"},
                 400: {"description": "Invalid data provided"},
                 409: {"description": "User with this username already exists"},
                 500: {"description": "Internal server error"},
             }
             )
def create_user(user_data: UserCreate, db: Session = Depends(get_session)):
    return crud.create_user(db=db, **user_data.dict())  # Распаковываем данные из схемы UserCreate


@router.get("/users/{user_id}", response_model=UserRead)  # Указываем, что возвращаемая модель - UserRead
def read_user(user_id: int, db: Session = Depends(get_session)):
    db_user = crud.get_user(db, user_id=user_id)
    if db_user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return db_user


@router.put("/users/{user_id}", response_model=UserRead)  # Указываем, что возвращаемая модель - UserRead
def update_user(user_id: int, user_data: UserUpdate, db: Session = Depends(get_session)):
    return crud.update_user(db=db, user_id=user_id, **user_data.dict(
        exclude_unset=True))  # Распаковываем данные из схемы UserUpdate, исключая пустые значения


@router.delete("/users/{user_id}", response_model=UserRead)  # Указываем, что возвращаемая модель - UserRead
def delete_user(user_id: int, db: Session = Depends(get_session)):
    return crud.delete_user(db=db, user_id=user_id)


@router.get("/users/", response_model=UserList)
def read_users(db: Session = Depends(get_session)):
    users = crud.get_all_users(db)
    return {"users": users}


```