# Flask-Prometheus-slack-Alert-Integration
This repository contains flask-app whose CPU metrics are being scrapped by Prometheus and metrics are visualised on Grafana and upon load/stress on CPU metrics, a slack alert gets triggered on the Workspace channel for DevOps/SRE Team

# How to run this project

1. Create a root directory named flask-app-prom and open any IDE like VS CODE  (SS-how it will look at the end)

![Screenshot from 2025-02-14 13-20-02](https://github.com/user-attachments/assets/f2c3aa86-91f9-4311-abfe-0c61ef5f2ccc)


```
$ mkdir flask-app-prom
```

## Project Requirements

2. Create requirements.txt file in the root directory with the following dependencies to run our flask application

```
Flask==3.1.0
psutil==7.0.0
python-dotenv==1.0.1
prometheus-client==0.21.1
```

3. Create app.py file in which our flask application will run which is just a simple flask application whose metrics we will monitor upon load

app.py file

```py
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

## 4. Create a dockerfile our application

```dockerfile
FROM python:3.8-slim

WORKDIR /app

COPY . /app

RUN pip install --no-cache-dir -r requirements.txt

EXPOSE 5000

ENV FLASK_APP=app.py

CMD ["python", "app.py"]
```

You can build,tag the dockerimage here and push it to any container repository like DockerHub, or AWS ECR etc.
We will build the docker image on the go when we spin up our containers using docker-compose in this project

## 5. Setting up Prometheus and Grafana 

Create a directory called prometheus inside root directory. This will have our prometheus and grafana related files to alerts,rules for alarm and alertmanager file
```sh
mkdir prometheus
```

Inside prometheus setup 3 seperate directories like alertmanager,rules and target (You can choose any name but i took for better easy convention and understanding)

```sh
mkdir alertmanager/
mkdir rules/
mkdir targets/
```

Inside alertmanager, create 2 yml files called alertmanager.yml and prometheus.yml

```yml
route:
  receiver: "slack-notifications"
  repeat_interval: 3h

receivers:
- name: "slack-notifications"
  slack_configs:
  - send_resolved: true
    api_url: "https://hooks.slack.com/services/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"
    channel: "#alert_channel"
    title: '🚨 [{{ .Status | toUpper }}] CPU Usage Alert'
    text: |
      *Alert:* High CPU Usage Detected!
      *Threshold:* Above 30%
      *Current Value:* {{ .CommonLabels.cpu_usage_percent }}%
      *Details:*
      {{ range .Alerts }}
        - *Instance:* {{ .Labels.instance }}
        - *Value:* {{ .Annotations.value }}%
        - *Description:* {{ .Annotations.description }}
      {{ end }}

```









