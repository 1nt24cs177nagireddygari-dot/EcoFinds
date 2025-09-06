#EcoFinds
# EcoFinds Backend

This is the backend for EcoFinds — a marketplace app built with FastAPI.

## Features

- User registration and login
- JWT-based authentication
- Product listing with category and keyword search
- Cart and purchase tracking
- SQLite database

## Run Locally

```bash
pip install fastapi uvicorn sqlalchemy pydantic jwt
python -m uvicorn main:app --reload
