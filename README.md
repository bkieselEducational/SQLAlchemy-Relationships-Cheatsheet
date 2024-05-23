# SQLAlchemy-Relationships-Cheatsheet

## Flask SQLAlchemy

### Attributes
```
back_populates -> The name of the relationship on the OTHER model which will be populated with instances of THIS class
secondary -> The join table to be used for the relationship
uselist=False -> Setting this to False tells sqlalchemy that when you access the relationship, you will return either a single class instance or None.
```

### Imports and Helpers
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

### One-to-One Relationship
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

### One-to-Many Relationship
```python
class Parent(db.Model):
    __tablename__ = 'parents'
    id = db.Column(db.Integer, primary_key=True)
    children = db.relationship("Child", back_populates="parent")

class Child(db.Model):
    __tablename__ = 'children'
    id = db.Column(db.Integer, primary_key=True)
    parent_id = db.Column(db.Integer, db.ForeignKey(add_prefix_for_prod('parents.id')))
    parent = db.relationship("Parent", back_populates="children")
```

### Many-to-Many Relationship
```python
# Creating the join table:
student_course_association = db.Table('student_course',
    db.Column('student_id', db.Integer, db.ForeignKey(add_prefix_for_prod('students.id'))),
    db.Column('course_id', db.Integer, db.ForeignKey(add_prefix_for_prod('courses.id')))
)

# Defining the models:
class Student(db.Model):
    __tablename__ = 'students'
    id = db.Column(db.Integer, primary_key=True)
    courses = db.relationship("Course", secondary=student_course_association, back_populates="students")

class Course(db.Model):
    __tablename__ = 'courses'
    id = db.Column(db.Integer, primary_key=True)
    students = db.relationship("Student", secondary=student_course_association, back_populates="courses")
```

### Seeding the JOIN Table

*Method 1*
```python
# Create instances of Student and Course
student1 = Student()
student2 = Student()
course1 = Course()
course2 = Course()

# Associate students with courses
student1.courses.append(course1)
student1.courses.append(course2)
student2.courses.append(course1)

# Add instances to the session and commit
db.session.add(student1)
db.session.add(student2)
db.session.add(course1)
db.session.add(course2)
db.session.commit()
```

*Method 2*
```python
values = [
  { 'student_id': 1, 'course_id': 1 },
  { 'student_id': 1, 'course_id': 2 },
  { 'student_id': 2, 'course_id': 1 },
]

db.session.execute(student_course_association.insert(), values)
db.session.commit()
```
