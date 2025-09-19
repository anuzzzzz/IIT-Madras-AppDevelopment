# IIT Madras App Development - Library Management System

## Project Overview

- **Institution:** Indian Institute of Technology Madras
- **Course:** Modern Application Development (MAD-2)
- **Type:** Full-stack web application
- **Tech Stack:** Flask + Vue.js + Redis + Celery
- **Purpose:** Multi-user e-book library management system

## Problem Statement

Design and develop a comprehensive library management system for issuing e-books to users with the following requirements:

### Core Features
- **Multi-user system:** Librarian (admin) and general users (students)
- **E-book management:** Issue, return, and access digital books
- **Section organization:** Categorized book collections
- **Search functionality:** Find books and sections efficiently
- **Background jobs:** Automated reports and notifications
- **Time-based access:** Automatic revocation after due dates

## Technical Architecture

### Backend Stack
- **Framework:** Flask (Python web framework)
- **Database:** SQLite with SQLAlchemy ORM
- **Caching:** Redis for performance optimization
- **Background Jobs:** Celery with Redis broker
- **Authentication:** Flask-Security with role-based access control (RBAC)

### Frontend Stack
- **Framework:** Vue.js 3 with CLI
- **Styling:** Bootstrap 5 (responsive design)
- **Templating:** Jinja2 (minimal usage)
- **API Communication:** Axios for REST API calls

### Infrastructure
- **Message Queue:** Redis + Celery for async tasks
- **File Storage:** Local filesystem for e-book content
- **Email Service:** SMTP for notifications
- **Deployment:** Development server (Flask built-in)

## System Features

### User Management
```python
# Role-based access control
class User(db.Model, UserMixin):
    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(255), unique=True, nullable=False)
    username = db.Column(db.String(255), unique=True, nullable=False)
    password = db.Column(db.String(255), nullable=False)
    active = db.Column(db.Boolean, default=True)
    roles = db.relationship('Role', secondary=roles_users, backref='users')
```

**User Types:**
- **Librarian:** Single admin user (pre-created)
- **General Users:** Students who can register and access books

### Section Management
```python
class Section(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    description = db.Column(db.Text)
    date_created = db.Column(db.DateTime, default=datetime.utcnow)
    books = db.relationship('Book', backref='section', lazy=True)
```

### E-book System
```python
class Book(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(200), nullable=False)
    content = db.Column(db.Text, nullable=False)
    author = db.Column(db.String(200), nullable=False)
    section_id = db.Column(db.Integer, db.ForeignKey('section.id'))
    date_created = db.Column(db.DateTime, default=datetime.utcnow)
```

### Book Issuance System
```python
class BookRequest(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'))
    book_id = db.Column(db.Integer, db.ForeignKey('book.id'))
    request_date = db.Column(db.DateTime, default=datetime.utcnow)
    issue_date = db.Column(db.DateTime)
    return_date = db.Column(db.DateTime)
    status = db.Column(db.String(20), default='pending')  # pending, approved, returned, revoked
```

## Key Functionalities

### User Features
1. **Registration & Authentication**
   - User signup with email/username
   - Secure login with password hashing
   - Profile management

2. **Book Discovery**
   - Browse sections and books
   - Search by title, author, or section
   - View book details and ratings

3. **Book Management**
   - Request books (max 5 concurrent)
   - Read issued books online
   - Return books before due date
   - Provide feedback and ratings

### Librarian Features
1. **Content Management**
   - Add/edit/delete sections
   - Add/edit/delete books
   - Assign books to sections

2. **User Management**
   - View all users and their activity
   - Issue books to users
   - Revoke book access
   - Monitor book status

3. **Analytics Dashboard**
   - View issued books statistics
   - User activity monitoring
   - Popular books/sections tracking

## Background Jobs & Automation

### Daily Reminder System
```python
@celery.task
def send_daily_reminders():
    """Send reminders to inactive users"""
    inactive_users = User.query.filter(
        User.last_login < datetime.utcnow() - timedelta(days=1)
    ).all()
    
    for user in inactive_users:
        send_email_reminder(user.email)
```

### Monthly Reports
```python
@celery.task
def generate_monthly_report():
    """Generate and email monthly activity reports"""
    users = User.query.filter(User.roles.any(Role.name == 'user')).all()
    
    for user in users:
        report_data = compile_user_activity(user.id)
        pdf_report = generate_pdf_report(report_data)
        send_email_with_attachment(user.email, pdf_report)
```

### Automatic Book Revocation
```python
@celery.task
def revoke_overdue_books():
    """Automatically revoke access for overdue books"""
    overdue_requests = BookRequest.query.filter(
        BookRequest.return_date < datetime.utcnow(),
        BookRequest.status == 'approved'
    ).all()
    
    for request in overdue_requests:
        request.status = 'revoked'
        db.session.commit()
```

## API Design

### RESTful Endpoints
```python
# Book management
GET    /api/books              # List all books
POST   /api/books              # Create new book (librarian)
GET    /api/books/<id>         # Get book details
PUT    /api/books/<id>         # Update book (librarian)
DELETE /api/books/<id>         # Delete book (librarian)

# User book requests
POST   /api/books/<id>/request # Request a book
POST   /api/books/<id>/return  # Return a book
GET    /api/users/books        # Get user's books

# Search functionality
GET    /api/search?q=<query>   # Search books and sections
```

### Frontend Integration
```javascript
// Vue.js component for book listing
export default {
  name: 'BookList',
  data() {
    return {
      books: [],
      searchQuery: '',
      loading: false
    }
  },
  methods: {
    async fetchBooks() {
      this.loading = true;
      try {
        const response = await axios.get('/api/books');
        this.books = response.data;
      } catch (error) {
        console.error('Error fetching books:', error);
      } finally {
        this.loading = false;
      }
    }
  }
}
```

## Security Features

### Authentication & Authorization
- **Password Hashing:** bcrypt for secure password storage
- **Session Management:** Flask-Session with Redis backend
- **CSRF Protection:** Built-in Flask-WTF protection
- **Role-based Access:** Decorator-based route protection

### Data Validation
```python
from flask_wtf import FlaskForm
from wtforms import StringField, TextAreaField, validators

class BookForm(FlaskForm):
    name = StringField('Book Name', [validators.Length(min=1, max=200)])
    author = StringField('Author', [validators.Length(min=1, max=200)])
    content = TextAreaField('Content', [validators.DataRequired()])
```

## Performance Optimizations

### Caching Strategy
```python
import redis
from flask import current_app

redis_client = redis.Redis(host='localhost', port=6379, db=0)

def get_popular_books():
    # Cache popular books for 1 hour
    cached_books = redis_client.get('popular_books')
    if cached_books:
        return json.loads(cached_books)
    
    books = Book.query.join(BookRequest).group_by(Book.id).order_by(
        func.count(BookRequest.id).desc()
    ).limit(10).all()
    
    redis_client.setex('popular_books', 3600, json.dumps(books))
    return books
```

### Database Optimizations
- Indexed search columns (book name, author)
- Efficient query patterns with SQLAlchemy
- Connection pooling for concurrent requests

## Demo Video

🎥 **[Watch Full Application Demo](https://drive.google.com/file/d/1dp5tQxfTKhQ_4bIz7O1uUK-uN9f7PHII/view?usp=drive_link)**

The demo video showcases:
- Complete user registration and authentication flow
- Librarian dashboard and book management
- User book request and reading experience
- Search functionality across sections and books
- Background job processing and email notifications
- Responsive design across different screen sizes

## Setup Instructions

### Prerequisites
```bash
# System requirements
- Python 3.7+
- Node.js 14+
- Redis server
- SQLite3

# Core dependencies (as installed)
Flask==2.3.3
Flask-SQLAlchemy==3.0.5
Flask-Security==3.0.0
Flask-CORS==4.0.1
Celery==5.2.7
Redis==5.0.8
email-validator==2.0.0
```

### Installation Steps
```bash
# 1. Clone repository
git clone https://github.com/anuzzzzz/IIT-Madras-AppDevelopment.git
cd IIT-Madras-AppDevelopment

# 2. Setup Python environment
python -m venv venv
source venv/bin/activate  # Linux/Mac
# venv\Scripts\activate   # Windows

# 3. Install dependencies
pip install -r requirements.txt

# 4. Setup frontend
cd frontend
npm install
npm run build

# 5. Initialize database
python init_db.py

# 6. Start Redis server
redis-server

# 7. Start Celery worker
celery -A app.celery worker --loglevel=info

# 8. Start application
python app.py
```

## Project Structure

Based on the installed dependencies and project structure:

```
IIT-Madras-AppDevelopment/
├── Backend/                       # Flask backend application
│   ├── app.py                     # Main Flask application
│   ├── app.log                    # Application logs
│   ├── config.py                  # Configuration settings
│   ├── models.py                  # Database models
│   ├── requirements.txt           # Python dependencies
│   ├── instance/                  # Instance-specific files
│   ├── migrations/                # Database migrations
│   ├── static/                    # Static assets
│   └── templates/                 # Jinja2 templates
├── Frontend/                      # Vue.js frontend
│   ├── src/                       # Vue source files
│   │   ├── components/            # Vue components
│   │   ├── views/                 # Page views
│   │   └── router/                # Vue router
│   ├── node_modules/              # Node dependencies
│   ├── package.json               # Frontend dependencies
│   └── dist/                      # Built frontend files
├── venv/                          # Python virtual environment
│   └── lib/python3.7/site-packages/  # Installed packages
├── redis.conf                     # Redis configuration
├── celerybeat-schedule            # Celery scheduler
└── README.md                      # Project documentation
```

### Key Dependencies Installed
- **Backend:** Flask 3.x, SQLAlchemy 2.x, Flask-Security 3.x, Celery 5.x, Redis 5.x
- **Frontend:** Vue.js 3, Bootstrap (via Popper.js), Axios for API calls
- **Database:** SQLite with Flask-SQLAlchemy ORM
- **Background Jobs:** Celery with Redis broker
- **Authentication:** Flask-Security with RBAC

## Key Learning Outcomes

### Technical Skills Demonstrated
1. **Full-stack Development:** End-to-end application development
2. **API Design:** RESTful service architecture
3. **Database Design:** Relational data modeling with SQLAlchemy
4. **Async Processing:** Background job implementation with Celery
5. **Caching Strategies:** Redis integration for performance
6. **Frontend Frameworks:** Modern Vue.js application development

### Software Engineering Practices
- **MVC Architecture:** Clean separation of concerns
- **Code Organization:** Modular project structure
- **Error Handling:** Comprehensive exception management
- **Security Best Practices:** Authentication and data validation
- **Testing:** Unit test coverage for critical components

## Deployment Considerations

### Production Readiness
- **Environment Configuration:** Separate dev/prod settings
- **Database Migration:** Alembic for schema changes
- **Logging:** Structured logging with rotation
- **Monitoring:** Health checks and performance metrics
- **Security:** HTTPS, secure headers, input sanitization

### Scalability Features
- **Horizontal Scaling:** Stateless application design
- **Caching Layer:** Redis for session and data caching
- **Background Processing:** Celery for heavy operations
- **Database Optimization:** Query optimization and indexing

---

*This project demonstrates comprehensive full-stack development skills with modern web technologies, focusing on real-world application requirements including user management, content delivery, and automated background processing.*
