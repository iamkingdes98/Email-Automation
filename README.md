# Email-Automation
Lifecation Email Automation 
- **`requirements.txt`**:
     ```
     Flask==2.0.1
     Flask-SQLAlchemy==2.5.1
     psycopg2-binary==2.9.1
     gunicorn==20.1.0
     sendgrid==6.9.7
     python-dotenv==0.19.0
     ```

   - **`render.yaml`**:
     ```
     services:
       - type: web
         name: email-campaign-automation
         env: python
         buildCommand: pip install -r requirements.txt
         startCommand: gunicorn app:app
         envVars:
           - key: PYTHON_VERSION
             value: 3.9.0
           - key: DATABASE_URL
             fromDatabase:
               name: email_campaign_db
               property: connectionString
           - key: FLASK_ENV
             value: production

     databases:
       - name: email_campaign_db
         databaseName: email_campaign
         user: email_campaign_user
     ```

   - **`.gitignore`**:
     ```
     __pycache__/
     *.py[cod]
     *$py.class
     *.so
     .env
     .venv
     env/
     venv/
     ENV/
     .DS_Store
     .env.local
     .env.development.local
     .env.test.local
     .env.production.local
     *.db
     ```

   - **`README.md`**:
     ```
     # Email Campaign Automation System

     An automated email campaign system built with Flask and SendGrid.

     ## Features
     - Automated client onboarding
     - Email template management
     - Campaign analytics
     - SendGrid integration

     ## Deployment Instructions

     ### Prerequisites
     1. Render account
     2. SendGrid account
     3. GitHub account

     ### Environment Variables
     Set the following in Render dashboard:
     - `DATABASE_URL`: Automatically set by Render
     - `SENDGRID_API_KEY`: Your SendGrid API key
     - `SENDGRID_FROM_EMAIL`: Verified sender email
     - `FLASK_ENV`: production

     ### Deploy to Render
     1. Fork/clone this repository
     2. Connect your GitHub account to Render
     3. Create a new Web Service
     4. Select this repository
     5. Render will automatically detect the configuration

     ## Local Development
     1. Clone the repository
     2. Create virtual environment: `python -m venv venv`
     3. Activate virtual environment
     4. Install dependencies: `pip install -r requirements.txt`
     5. Copy `.env.example` to `.env` and fill in values
     6. Run: `flask run`
     ```
