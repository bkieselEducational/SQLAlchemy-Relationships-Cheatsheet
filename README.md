# SQLAlchemy Relationships Cheatsheet

## Flask SQLAlchemy

Let us first consider some things about SQL relationships and thier intersection with an ORM. Ultimately, anything we can do through defining a relationship on a model can be accomplished without defining that relationship, by using the foreign keys we define in our models (ultimately tables). The use of the relationship is a convenience feature and we absolutely should use it, but we need to keep in mind that at the end of the day, keying into that relationship is no different than sending a raw SQL query (or series of queries) to the database ourselves. 

## A Note about Cascade Deletes:

One thing that can be a little unintuitive about setting this property is that when we set it on a foreign key, we are of course setting it on the 'child' side of that relationship, wether it be a 1-to-1 or a 1-to-many relationship. When we set this property an a relationship, it must always be set on the 1 side that is the 'parent'. In fact, for this reason, if we try to set this property on one of the sides of a many-to-many relationship, sqlalchemy will throw an error. This is a little bit of a gotcha! Technically, we cannot set a cascade delete on a many-to-many relationship in sqlalchemy. To get the behavior we desire, we must set this property on the Foreign Keys defined in the Join table that we use to implement the many-to-many. In this sense, the deletion is functioning as the 'many' side of a 1-to-many relationship. 

### Attributes
```
back_populates -> The name of the relationship on the OTHER model which will be populated with instances of THIS class.
cascade="delete" -> We want to set this property to ensure that when we delete the row, all rows in other tables that were referencing it through a foreign key are also deleted.
secondary -> The join table to be used for the relationship.
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
In this example a User can HAVE 1 Profile and a Profile can BELONG to 1 User.
```python

class User(db.Model):
    __tablename__ = 'users'
    id = db.Column(db.Integer, primary_key=True)
    profile = db.relationship("Profile", uselist=False, back_populates="user", cascade="delete") # uselist=False makes it 1-to-1
# Note that the cascade delete is set on the parent of a 1-to-1. If we did not have such a relationship, we could implement this by setting the cascade delete on the Foreign Key definition.

class Profile(db.Model):
    __tablename__ = 'profiles'
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey(add_prefix_for_prod('users.id'), unique=True))
    user = db.relationship("User", back_populates="profile")
```

### One-to-Many Relationship
In this example a Parent can HAVE many children, but a Child can BELONG to only 1 Parent.
```python
class Parent(db.Model):
    __tablename__ = 'parents'
    id = db.Column(db.Integer, primary_key=True)
    children = db.relationship("Child", back_populates="parent", cascade="delete")
# Note that the cascade delete is set on the parent of a 1-to-many.

class Child(db.Model):
    __tablename__ = 'children'
    id = db.Column(db.Integer, primary_key=True)
    parent_id = db.Column(db.Integer, db.ForeignKey(add_prefix_for_prod('parents.id')))
    parent = db.relationship("Parent", back_populates="children")
```

### Many-to-Many Relationship
In this example A Student can HAVE many Courses and a Course can also BELONG to many Students.
Another key point here is that the cascade delete needs to be set on the Foreign Keys themselves, NOT the relationship!
```python
# Creating the join table:
student_course_association = db.Table('student_course',
    db.Column('student_id', db.Integer, db.ForeignKey(add_prefix_for_prod('students.id', ondelete='CASCADE'))),
    db.Column('course_id', db.Integer, db.ForeignKey(add_prefix_for_prod('courses.id', ondelete='CASCADE')))
)

# Defining the models:
class Student(db.Model):
    __tablename__ = 'students'
    id = db.Column(db.Integer, primary_key=True)
    courses = db.relationship("Course", secondary=student_course_association, back_populates="students")
# Attempting to add the cascade delete to this relationship will throw an error.

class Course(db.Model):
    __tablename__ = 'courses'
    id = db.Column(db.Integer, primary_key=True)
    students = db.relationship("Student", secondary=student_course_association, back_populates="courses")
# Attempting to add the cascade delete to this relationship will throw an error.
```

### Seeding the JOIN Table (from example above)

*Method 1: Verbose*
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

*Method 2: Succinct*
```python
values = [
  { 'student_id': 1, 'course_id': 1 },
  { 'student_id': 1, 'course_id': 2 },
  { 'student_id': 2, 'course_id': 1 }
]

db.session.execute(student_course_association.insert(), values)
db.session.commit()
```
