# SQLAlchemy-Relationships-Cheatsheet

## Flask SQLAlchemy

### Imports
```python
import os
SCHEMA = os.getenv('SCHEMA')
environment = os.getenv('FLASK_ENV')

from flask_sqlalchemy import SQLAlchemy
db = SQLAlchemy()

def add_prefix_for_prod(attr):
    if environment == "production":
        return f"{SCHEMA}.{attr}"
    else:
        return attr
```

### 1-to-1 Relationship
```python

class User(db.Model):
    __tablename__ = 'users'
    id = db.Column(db.Integer, primary_key=True)
    profile = db.relationship("Profile", uselist=False, back_populates="user") # uselist=False makes it 1-to-1

class Profile(db.Model):
    __tablename__ = 'profiles'
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey(add_prefix_for_prod('users.id')))
    user = db.relationship("User", back_populates="profile")
```
