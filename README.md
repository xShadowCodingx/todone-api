# SCHEMA

'''

your_todo_app/
├── instance/
│   └── config.py  # Sensitive configuration (not version controlled)
├── todoapp/
│   ├── __init__.py      # Application factory and app setup
│   ├── models.py        # Database models (e.g., Todo, User)
│   ├── config.py        # Default configuration
│   ├── auth.py          # Authentication blueprint
│   ├── main.py          # Main to-do list blueprint
│   ├── templates/
│   │   ├── base.html
│   │   ├── auth/
│   │   │   └── login.html
│   │   └── main/
│   │       └── index.html
│   └── static/
│       ├── css/
│       └── js/
├── tests/
│   ├── conftest.py      # Pytest fixtures
│   └── test_todo.py     # Unit tests
├── venv/                # Virtual environment
├── .env                 # Environment variables (for development)
├── .flaskenv            # Flask environment variables (e.g., FLASK_APP)
└── requirements.txt     # Project dependencies

'''

Core Components of the Application Factory
Here's how to implement the key parts:

1. todoapp/__init__.py (The Application Factory)
This file will contain your create_app function, which is the heart of your application factory.

'''
import os
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_login import LoginManager

# Initialize extensions outside the factory
db = SQLAlchemy()
migrate = Migrate()
login_manager = LoginManager()
login_manager.login_view = 'auth.login' # Where to redirect if user is not logged in

def create_app(test_config=None):
    """
    Application factory function.
    Initializes and returns a Flask application instance.
    """
    app = Flask(__name__, instance_relative_config=True)

    # Configure the application
    app.config.from_object('todoapp.config.Config')

    if test_config:
        # Load the test configuration if passed in
        app.config.from_mapping(test_config)
    else:
        # Load the instance config, if it exists, when not testing
        app.config.from_pyfile('config.py', silent=True)

    # Ensure the instance folder exists
    try:
        os.makedirs(app.instance_path)
    except OSError:
        pass

    # Initialize extensions with the app
    db.init_app(app)
    migrate.init_app(app, db)
    login_manager.init_app(app)

    # Register Blueprints
    from .auth import auth_bp
    app.register_blueprint(auth_bp)

    from .main import main_bp
    app.register_blueprint(main_bp)

    # You might want to define a user loader for Flask-Login here or in models.py
    # For example:
    from .models import User

    @login_manager.user_loader
    def load_user(user_id):
        return User.query.get(int(user_id))

    # A simple route for demonstration (can be removed later)
    @app.route('/hello')
    def hello():
        return 'Hello, World!'

    return app
'''

2. todoapp/config.py (Default Configuration)
This file holds default configuration settings. Sensitive information should be loaded from environment variables or instance/config.py.

'''
import os

class Config:
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'a-very-secret-key-that-should-be-stronger-in-production'
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL') or 'sqlite:///' + os.path.join(os.path.abspath(os.path.dirname(__file__)), 'app.db')
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    # Add other configurations like email server, admin users, etc.
'''

3. instance/config.py (Production/Sensitive Configuration)
This file is not checked into version control (add it to .gitignore) and holds sensitive or environment-specific configurations.

'''
# instance/config.py (Add to .gitignore)
SECRET_KEY = 'your-production-secret-key-generated-randomly'
SQLALCHEMY_DATABASE_URI = 'postgresql://user:password@localhost/prod_db'
# Other production-specific settings
'''

4. todoapp/models.py (Database Models)
Define your SQLAlchemy models here.

'''
from todoapp import db
from datetime import datetime
from flask_login import UserMixin

class User(UserMixin, db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(64), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(128))
    todos = db.relationship('Todo', backref='author', lazy='dynamic')

    def __repr__(self):
        return f'<User {self.username}>'

    # Add methods for password hashing (e.g., using werkzeug.security)

class Todo(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(120), nullable=False)
    description = db.Column(db.Text)
    complete = db.Column(db.Boolean, default=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'))

    def __repr__(self):
        return f'<Todo {self.title}>'
'''

5. todoapp/auth.py (Authentication Blueprint)
This blueprint handles user authentication (registration, login, logout).

'''
from flask import Blueprint, render_template, request, redirect, url_for, flash
from flask_login import login_user, logout_user, current_user, login_required
from todoapp import db # Assuming db is initialized in __init__.py
from todoapp.models import User # Assuming User model is in models.py
# You'll need to add forms (e.g., using Flask-WTF) and password hashing

auth_bp = Blueprint('auth', __name__, url_prefix='/auth')

@auth_bp.route('/register', methods=['GET', 'POST'])
def register():
    # Implement registration logic here
    # Use forms for input validation
    # Hash passwords before saving
    if request.method == 'POST':
        username = request.form.get('username')
        email = request.form.get('email')
        password = request.form.get('password')

        user = User.query.filter_by(username=username).first()
        if user:
            flash('Username already exists!')
            return redirect(url_for('auth.register'))

        # Add password hashing here
        new_user = User(username=username, email=email, password_hash="hashed_password") # Replace with actual hash
        db.session.add(new_user)
        db.session.commit()
        flash('Registration successful! Please log in.')
        return redirect(url_for('auth.login'))
    return render_template('auth/register.html')

@auth_bp.route('/login', methods=['GET', 'POST'])
def login():
    # Implement login logic here
    # Authenticate user and log them in using login_user
    if request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')
        user = User.query.filter_by(username=username).first()

        if user and "check_password_hash_here_with_user_password_hash_and_provided_password": # Replace with actual check
            login_user(user)
            flash('Logged in successfully!')
            next_page = request.args.get('next')
            return redirect(next_page or url_for('main.index'))
        else:
            flash('Invalid username or password.')
    return render_template('auth/login.html')

@auth_bp.route('/logout')
@login_required
def logout():
    logout_user()
    flash('You have been logged out.')
    return redirect(url_for('auth.login'))
'''

6. todoapp/main.py (Main To-Do List Blueprint)
This blueprint handles the core to-do list functionality.

'''
from flask import Blueprint, render_template, request, redirect, url_for, flash
from flask_login import login_required, current_user
from todoapp import db
from todoapp.models import Todo

main_bp = Blueprint('main', __name__)

@main_bp.route('/')
@login_required
def index():
    todos = current_user.todos.order_by(Todo.created_at.desc()).all()
    return render_template('main/index.html', todos=todos)

@main_bp.route('/add', methods=['POST'])
@login_required
def add_todo():
    title = request.form.get('title')
    if not title:
        flash('Title cannot be empty!', 'error')
        return redirect(url_for('main.index'))
    new_todo = Todo(title=title, user_id=current_user.id)
    db.session.add(new_todo)
    db.session.commit()
    flash('Todo added successfully!')
    return redirect(url_for('main.index'))

@main_bp.route('/update/<int:todo_id>')
@login_required
def update_todo(todo_id):
    todo = Todo.query.filter_by(id=todo_id, user_id=current_user.id).first_or_404()
    todo.complete = not todo.complete
    db.session.commit()
    flash('Todo updated!')
    return redirect(url_for('main.index'))

@main_bp.route('/delete/<int:todo_id>')
@login_required
def delete_todo(todo_id):
    todo = Todo.query.filter_by(id=todo_id, user_id=current_user.id).first_or_404()
    db.session.delete(todo)
    db.session.commit()
    flash('Todo deleted!')
    return redirect(url_for('main.index'))
'''

7. templates/ and static/
Organize your Jinja2 templates and static files (CSS, JS, images) within their respective blueprint directories for better organization, or have a general templates and static folder for shared resources.

templates/base.html: Your base layout.
templates/auth/register.html, templates/auth/login.html: Forms for authentication.
templates/main/index.html: The main to-do list display.

8. wsgi.py (Entry Point for Production)
For production deployment, you'll use this file to serve your application.

'''
from todoapp import create_app

app = create_app()

if __name__ == '__main__':
    app.run(debug=True)
'''

Running Your Application
Development
Set up Virtual Environment:
Bash

python -m venv venv
source venv/bin/activate # On Windows: .\venv\Scripts\activate
Install Dependencies:
Bash

pip install Flask Flask-SQLAlchemy Flask-Migrate Flask-Login python-dotenv # Add Flask-WTF, Werkzeug for password hashing
pip freeze > requirements.txt
Create .flaskenv and .env: .flaskenv:
FLASK_APP=wsgi.py
FLASK_ENV=development
.env (for development secrets):
SECRET_KEY='your-dev-secret-key'
DATABASE_URL='sqlite:///instance/dev.db'
Initialize Database:
Bash

flask db init
flask db migrate -m "Initial migration"
flask db upgrade
Run the App:
Bash

flask run
(Or python wsgi.py if you prefer, but flask run is often more convenient for development with the factory.)
Production
For production, you'd use a WSGI server like Gunicorn or Waitress.

Bash

# Install Gunicorn
pip install gunicorn

# Run with Gunicorn (assuming wsgi.py is your entry point)
gunicorn --workers 4 'wsgi:app'

## Key Best Practices
Separate Concerns: Use Blueprints to logically separate parts of your application (e.g., authentication, user management, admin, API).
Externalize Configuration: Keep sensitive information (like SECRET_KEY and database credentials) out of your main codebase using environment variables (.env, .flaskenv) and the instance folder.
Lazy Extension Initialization: Initialize Flask extensions (like SQLAlchemy, Migrate, LoginManager) outside the create_app function but bind them inside it using extension.init_app(app). This allows you to have multiple app instances.
Database Migrations: Use Flask-Migrate (or Alembic directly) to manage your database schema changes.
Testing: Design your factory to accept a test_config parameter, making it easy to configure your app for testing with an in-memory database or a separate test database.
Error Handling: Implement custom error pages (e.g., for 404, 500 errors).
Forms and Validation: Use Flask-WTF for robust form handling and validation.
Password Security: Always hash and salt user passwords (e.g., using Werkzeug.security.generate_password_hash and check_password_hash).
Logging: Configure logging for your application to help debug issues in development and production.