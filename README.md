# Flask-Prometheus-slack-Alert-Integration
This repository contains flask-app whose CPU metrics are being scrapped by Prometheus and metrics are visualised on Grafana and upon load/stress on CPU metrics, a slack alert gets triggered on the Workspace channel for DevOps/SRE Team

# How to run this project

1. Create a root directory named flask-app-prom and open any IDE like VS CODE  

```
$ mkdir flask-app-prom
```

# Project Requirements

2. Create requirements.txt file in the root directory with the following dependencies to run our flask application

```
Flask==3.1.0
psutil==7.0.0
python-dotenv==1.0.1
prometheus-client==0.21.1
```

3. Create app.py file in which our flask application will run which is just a simple flask application whose metrics we will monitor upon load

app.py file
```
from flask import Flask, request, abort
import threading
import time
from dotenv import load_dotenv
import psutil,os
from prometheus_client import Gauge, generate_latest, CONTENT_TYPE_LATEST

try:
    load_dotenv()
except Exception as e:
    print(f"Error loading .env file: {e}")

app = Flask(__name__)

# Prometheus Metrics
cpu_usage = Gauge('cpu_usage_percent', 'CPU usage percentage')

def monitor_cpu_usage():
    while True:
        try:
            usage = psutil.cpu_percent()
            cpu_usage.set(usage)
            print(f"CPU Usage Updated: {usage}%")
        except Exception as e:
            print(f"Error in monitoring thread: {e}")
            time.sleep(5)  

@app.route('/')
def home():
    return "Flask CPU Metrics App Running!"

@app.route('/metrics')
def metrics():
    try:
        return generate_latest(), 200, {'Content-Type': CONTENT_TYPE_LATEST}
    except Exception as e:
        return str(e), 500

def main():
    monitor_thread = threading.Thread(target=monitor_cpu_usage, daemon=True)
    monitor_thread.start()
    app.run(host='0.0.0.0', port=5000, debug=False)

if __name__ == '__main__':
    main()
```

4. Create a dockerfile our application

```
FROM python:3.8-slim

WORKDIR /app

COPY . /app

RUN pip install --no-cache-dir -r requirements.txt

EXPOSE 5000

ENV FLASK_APP=app.py

CMD ["python", "app.py"]
```
