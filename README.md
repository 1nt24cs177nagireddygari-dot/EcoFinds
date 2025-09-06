from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from typing import List, Optional
from sqlalchemy import create_engine, Column, Integer, String, Float, ForeignKey, Table
from sqlalchemy.orm import sessionmaker, declarative_base, relationship, Session
import hashlib, jwt

# --- Config ---
SECRET = "ecofinds_secret"
DATABASE_URL = "sqlite:///./ecofinds.db"

# --- DB Setup ---
engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(bind=engine, autoflush=False)
Base = declarative_base()

# --- Models ---
cart_table = Table("cart", Base.metadata,
    Column("user_id", Integer, ForeignKey("users.id")),
    Column("product_id", Integer, ForeignKey("products.id"))
)

purchase_table = Table("purchases", Base.metadata,
    Column("user_id", Integer, ForeignKey("users.id")),
    Column("product_id", Integer, ForeignKey("products.id"))
)

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    email = Column(String, unique=True)
    username = Column(String)
    password = Column(String)
    products = relationship("Product", back_populates="owner")
    cart = relationship("Product", secondary=cart_table)
    purchases = relationship("Product", secondary=purchase_table)

class Product(Base):
    __tablename__ = "products"
    id = Column(Integer, primary_key=True)
    title = Column(String)
    description = Column(String)
    category = Column(String)
    price = Column(Float)
    image_url = Column(String)
    owner_id = Column(Integer, ForeignKey("users.id"))
    owner = relationship("User", back_populates="products")

Base.metadata.create_all(bind=engine)

# --- Schemas ---
class UserCreate(BaseModel):
    email: str
    username: str
    password: str

class UserLogin(BaseModel):
    email: str
    password: str

class UserOut(BaseModel):
    id: int
    email: str
    username: str
    class Config:
        orm_mode = True

class ProductCreate(BaseModel):
    title: str
    description: str
    category: str
    price: float
    image_url: str

class ProductOut(ProductCreate):
    id: int
    owner_id: int
    class Config:
        orm_mode = True

# --- Utils ---
def hash_password(password: str) -> str:
    return hashlib.sha256(password.encode()).hexdigest()

def verify_password(password: str, hashed: str) -> bool:
    return hash_password(password) == hashed

def create_token(data: dict) -> str:
    return jwt.encode(data, SECRET, algorithm="HS256")

def decode_token(token: str) -> int:
    return jwt.decode(token, SECRET, algorithms=["HS256"])["user_id"]

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# --- App ---
app = FastAPI(title="EcoFinds")

@app.get("/")
def read_root():
    return {"message": "EcoFinds is live!"}

# --- Auth ---
@app.post("/auth/register", response_model=UserOut)
def register(user: UserCreate, db: Session = Depends(get_db)):
    hashed_pw = hash_password(user.password)
    db_user = User(email=user.email, username=user.username, password=hashed_pw)
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user

@app.post("/auth/login")
def login(user: UserLogin, db: Session = Depends(get_db)):
    db_user = db.query(User).filter(User.email == user.email).first()
    if not db_user or not verify_password(user.password, db_user.password):
        raise HTTPException(status_code=401, detail="Invalid credentials")
    token = create_token({"user_id": db_user.id})
    return {"access_token": token}

@app.get("/auth/me", response_model=UserOut)
def get_profile(token: str, db: Session = Depends(get_db)):
    user_id = decode_token(token)
    user = db.query(User).get(user_id)
    return user

# --- Products ---
@app.post("/products/", response_model=ProductOut)
def create_product(product: ProductCreate, token: str, db: Session = Depends(get_db)):
    user_id = decode_token(token)
    db_product = Product(**product.dict(), owner_id=user_id)
    db.add(db_product)
    db.commit()
    db.refresh(db_product)
    return db_product

@app.get("/products/", response_model=List[ProductOut])
def list_products(category: Optional[str] = None, keyword: Optional[str] = None, db: Session = Depends(get_db)):
    query = db.query(Product)
    if category:
        query = query.filter(Product.category == category)
    if keyword:
        query = query.filter(Product.title.contains(keyword))
    return query.all()

@app.get("/products/{product_id}", response_model=ProductOut)
def get_product(product_id: int, db: Session = Depends(get_db)):
    product = db.query(Product).get(product_id)
    if not product:
        raise HTTPException(status_code=404, detail="Product not found")
    return product

@app.put("/products/{product_id}", response_model=ProductOut)
def update_product(product_id: int, product: ProductCreate, token: str, db: Session = Depends(get_db)):
    user_id = decode_token(token)
    db_product = db.query(Product).get(product_id)
    if db_product.owner_id != user_id:
        raise HTTPException(status_code=403, detail="Not authorized")
    for key, value in product.dict().items():
        setattr(db_product, key, value)
    db.commit()
    db.refresh(db_product)
    return db_product

@app.delete("/products/{product_id}")
def delete_product(product_id: int, token: str, db: Session = Depends(get_db)):
    user_id = decode_token(token)
    db_product = db.query(Product).get(product_id)
    if db_product.owner_id != user_id:
        raise HTTPException(status_code=403, detail="Not authorized")
    db.delete(db_product)
    db.commit()
    return {"message": "Deleted"}

# --- Cart ---
@app.post("/cart/add/{product_id}")
def add_to_cart(product_id: int, token: str, db: Session = Depends(get_db)):
    user_id = decode_token(token)
    user = db.query(User).get(user_id)
    product = db.query(Product).get(product_id)
    user.cart.append(product)
    db.commit()
    return {"message": "Added to cart"}

@app.get("/cart/", response_model=List[ProductOut])
def view_cart(token: str, db: Session = Depends(get_db)):
    user_id = decode_token(token)
    user = db.query(User).get(user_id)
    return user.cart

@app.post("/cart/checkout")
def checkout(token: str, db: Session = Depends(get_db)):
    user_id = decode_token(token)
    user = db.query(User).get(user_id)
    user.purchases.extend(user.cart)
    user.cart.clear()
    db.commit()
    return {"message": "Purchase complete"}

@app.get("/cart/purchases", response_model=List[ProductOut])
def view_purchases(token: str, db: Session = Depends(get_db)):
    user_id = decode_token(token)
    user = db.query(User).get(user_id)
    return user.purchases
