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


import os
from flask import Flask, render_template, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime
from sendgrid import SendGridAPIClient
from sendgrid.helpers.mail import Mail
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

app = Flask(__name__)

# Database configuration
database_url = os.getenv('DATABASE_URL')
if database_url and database_url.startswith('postgres://'):
    database_url = database_url.replace('postgres://', 'postgresql://', 1)

app.config['SQLALCHEMY_DATABASE_URI'] = database_url or 'sqlite:///database/onboarding.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

# Models
class Client(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    business_name = db.Column(db.String(100), nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    industry = db.Column(db.String(50), nullable=False)
    target_audience = db.Column(db.String(200))
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    campaign_status = db.Column(db.String(20), default='pending')

class EmailTemplate(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    industry = db.Column(db.String(50), nullable=False)
    template_type = db.Column(db.String(50), nullable=False)
    subject = db.Column(db.String(200), nullable=False)
    content = db.Column(db.Text, nullable=False)

class CampaignLog(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    client_id = db.Column(db.Integer, db.ForeignKey('client.id'), nullable=False)
    email_subject = db.Column(db.String(200), nullable=False)
    sent_at = db.Column(db.DateTime, default=datetime.utcnow)
    status = db.Column(db.String(20), default='sent')

def send_email(client, template):
    """Send an email using SendGrid"""
    try:
        content = template.content.format(
            business_name=client.business_name,
            target_audience=client.target_audience
        )

        message = Mail(
            from_email=os.getenv('SENDGRID_FROM_EMAIL'),
            to_emails=client.email,
            subject=template.subject,
            html_content=content
        )

        sg = SendGridAPIClient(os.getenv('SENDGRID_API_KEY'))
        response = sg.send(message)

        # Log the campaign
        log = CampaignLog(
            client_id=client.id,
            email_subject=template.subject,
            status='sent' if response.status_code == 202 else 'failed'
        )
        db.session.add(log)
        db.session.commit()

        return True
    except Exception as e:
        print(f"Error sending email: {str(e)}")
        return False

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/submit', methods=['POST'])
def submit():
    try:
        data = request.json
        new_client = Client(
            business_name=data['business_name'],
            email=data['email'],
            industry=data['industry'],
            target_audience=data['target_audience']
        )

        db.session.add(new_client)
        db.session.commit()

        # Send welcome email
        template = EmailTemplate.query.filter_by(
            industry=new_client.industry,
            template_type='welcome'
        ).first() or EmailTemplate.query.filter_by(
            industry='default',
            template_type='welcome'
        ).first()

        if template:
            email_sent = send_email(new_client, template)
            return jsonify({
                'status': 'success',
                'email_sent': email_sent
            })

        return jsonify({'status': 'success', 'email_sent': False})

    except Exception as e:
        return jsonify({'status': 'error', 'message': str(e)})

@app.route('/analytics')
def analytics():
    logs = CampaignLog.query.all()
    analytics_data = [{
        'client_id': log.client_id,
        'email_subject': log.email_subject,
        'sent_at': log.sent_at.strftime('%Y-%m-%d %H:%M:%S'),
        'status': log.status
    } for log in logs]

    return jsonify(analytics_data)

if __name__ == '__main__':
    with app.app_context():
        db.create_all()
    app.run(debug=False)