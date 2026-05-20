# 3. Scripting for Automation (Python)

## Table of Contents
- [Python Installation & Setup](#python-installation--setup)
- [Python Fundamentals](#python-fundamentals)
- [File I/O & OS Module](#file-io--os-module)
- [Subprocess Module](#subprocess-module)
- [Working with APIs (Requests)](#working-with-apis-requests)
- [JSON & YAML Parsing](#json--yaml-parsing)
- [Error Handling](#error-handling)
- [Projects](#projects)

---

## Python Installation & Setup

```bash
# Check if Python is installed
python3 --version
# Output: Python 3.10.12

# Install Python (Ubuntu)
sudo apt update
sudo apt install -y python3 python3-pip python3-venv

# Create a virtual environment (best practice)
mkdir ~/python-devops && cd ~/python-devops
python3 -m venv venv

# Activate virtual environment
source venv/bin/activate
# Prompt changes to: (venv) student@devops:~/python-devops$

# Install common packages
pip install requests pyyaml paramiko boto3 python-dotenv

# Save dependencies
pip freeze > requirements.txt

# Install from requirements
pip install -r requirements.txt

# Deactivate virtual environment
deactivate
```

---

## Python Fundamentals

### Variables & Data Types

```python
# strings
name = "DevOps Engineer"
server = 'web-01'

# numbers
port = 8080
cpu_usage = 78.5

# booleans
is_running = True
has_errors = False

# lists (ordered, mutable)
servers = ["web-01", "web-02", "db-01", "db-02"]
servers.append("cache-01")
print(servers[0])      # web-01
print(servers[-1])     # cache-01
print(len(servers))    # 5

# dictionaries (key-value pairs)
server_config = {
    "hostname": "web-01",
    "ip": "192.168.1.10",
    "port": 80,
    "services": ["nginx", "node"],
    "active": True
}
print(server_config["hostname"])   # web-01
server_config["region"] = "us-east-1"  # Add new key

# tuples (ordered, immutable)
coordinates = (40.7128, -74.0060)

# sets (unique values)
active_ports = {80, 443, 8080, 80}  # {80, 443, 8080}
```

### Control Flow

```python
# If-elif-else
status_code = 503

if status_code == 200:
    print("OK")
elif status_code == 404:
    print("Not Found")
elif status_code >= 500:
    print(f"Server Error: {status_code}")
else:
    print(f"Unknown status: {status_code}")

# For loops
servers = ["web-01", "web-02", "db-01"]
for server in servers:
    print(f"Checking {server}...")

# For with index
for i, server in enumerate(servers):
    print(f"{i+1}. {server}")

# Range
for i in range(5):
    print(f"Attempt {i+1}")

# While loop
retries = 0
max_retries = 3
while retries < max_retries:
    print(f"Retry {retries + 1}/{max_retries}")
    retries += 1

# List comprehension
ports = [80, 443, 8080, 3000, 5000]
high_ports = [p for p in ports if p > 1024]
# [3000, 5000]
```

### Functions

```python
def check_server(hostname, port=80, timeout=5):
    """Check if a server is reachable."""
    import socket
    try:
        sock = socket.create_connection((hostname, port), timeout)
        sock.close()
        return True
    except (socket.timeout, ConnectionRefusedError, OSError):
        return False

# Usage
result = check_server("google.com", 443)
print(f"Server reachable: {result}")

# Function with multiple returns
def get_system_info():
    import platform
    return {
        "os": platform.system(),
        "version": platform.version(),
        "machine": platform.machine(),
        "hostname": platform.node()
    }

info = get_system_info()
print(info)
```

### String Formatting

```python
hostname = "web-01"
ip = "192.168.1.10"
cpu = 78.5

# f-strings (recommended)
print(f"Server {hostname} ({ip}) - CPU: {cpu:.1f}%")

# Multi-line strings
config = f"""
server {{
    listen 80;
    server_name {hostname};
    root /var/www/html;
}}
"""
print(config)
```

---

## File I/O & OS Module

### Reading & Writing Files

```python
# Write to a file
with open("server_list.txt", "w") as f:
    f.write("web-01\n")
    f.write("web-02\n")
    f.write("db-01\n")

# Read entire file
with open("server_list.txt", "r") as f:
    content = f.read()
    print(content)

# Read line by line
with open("server_list.txt", "r") as f:
    for line in f:
        server = line.strip()
        print(f"Processing: {server}")

# Read all lines into a list
with open("server_list.txt", "r") as f:
    servers = [line.strip() for line in f.readlines()]
    print(servers)
    # ['web-01', 'web-02', 'db-01']

# Append to file
with open("server_list.txt", "a") as f:
    f.write("cache-01\n")
```

### OS Module

```python
import os

# Current working directory
cwd = os.getcwd()
print(f"Current directory: {cwd}")

# List directory contents
files = os.listdir("/var/log")
print(files)

# Check if file/directory exists
print(os.path.exists("/etc/nginx/nginx.conf"))  # True/False
print(os.path.isfile("/etc/hosts"))              # True
print(os.path.isdir("/var/log"))                 # True

# Create directories
os.makedirs("backups/2024/january", exist_ok=True)

# Get file size
size = os.path.getsize("/var/log/syslog")
print(f"Syslog size: {size / 1024:.2f} KB")

# Walk through directory tree
for root, dirs, files in os.walk("/var/log"):
    for file in files:
        filepath = os.path.join(root, file)
        size = os.path.getsize(filepath)
        if size > 1024 * 1024:  # > 1MB
            print(f"Large file: {filepath} ({size / 1024 / 1024:.2f} MB)")

# Environment variables
home = os.environ.get("HOME")
path = os.environ.get("PATH")
db_password = os.environ.get("DB_PASSWORD", "default_value")
print(f"Home: {home}")

# Set environment variable
os.environ["APP_ENV"] = "production"

# Path manipulation
filepath = "/var/log/nginx/access.log"
print(os.path.dirname(filepath))    # /var/log/nginx
print(os.path.basename(filepath))   # access.log
print(os.path.splitext(filepath))   # ('/var/log/nginx/access', '.log')
```

### shutil Module

```python
import shutil

# Copy file
shutil.copy("source.txt", "destination.txt")

# Copy directory
shutil.copytree("project/", "project_backup/")

# Move/rename
shutil.move("old_name.txt", "new_name.txt")

# Remove directory tree
shutil.rmtree("temp_directory/")

# Get disk usage
usage = shutil.disk_usage("/")
print(f"Total: {usage.total / (1024**3):.1f} GB")
print(f"Used:  {usage.used / (1024**3):.1f} GB")
print(f"Free:  {usage.free / (1024**3):.1f} GB")
```

---

## Subprocess Module

```python
import subprocess

# Run a simple command
result = subprocess.run(["ls", "-la", "/var/log"], capture_output=True, text=True)
print("STDOUT:", result.stdout)
print("STDERR:", result.stderr)
print("Return code:", result.returncode)

# Run command with shell=True (use when you need pipes)
result = subprocess.run(
    "ps aux | grep nginx | wc -l",
    shell=True,
    capture_output=True,
    text=True
)
print(f"Nginx processes: {result.stdout.strip()}")

# Check command success
result = subprocess.run(["systemctl", "is-active", "nginx"], capture_output=True, text=True)
if result.returncode == 0:
    print("Nginx is running")
else:
    print("Nginx is NOT running")

# Run with timeout
try:
    result = subprocess.run(
        ["ping", "-c", "3", "google.com"],
        capture_output=True, text=True,
        timeout=10
    )
    print(result.stdout)
except subprocess.TimeoutExpired:
    print("Command timed out!")

# Run multiple commands sequentially
commands = [
    ["sudo", "apt", "update"],
    ["sudo", "apt", "install", "-y", "nginx"],
    ["sudo", "systemctl", "start", "nginx"],
    ["sudo", "systemctl", "enable", "nginx"],
]

for cmd in commands:
    print(f"Running: {' '.join(cmd)}")
    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.returncode != 0:
        print(f"ERROR: {result.stderr}")
        break
    print(f"OK: {result.stdout[:100]}")
```

---

## Working with APIs (Requests)

### Installation & Basic Usage

```bash
pip install requests
```

```python
import requests

# GET request
response = requests.get("https://api.github.com/users/torvalds")
print(f"Status: {response.status_code}")
print(f"Content-Type: {response.headers['content-type']}")

# Parse JSON response
data = response.json()
print(f"Name: {data['name']}")
print(f"Public repos: {data['public_repos']}")
print(f"Followers: {data['followers']}")

# GET with query parameters
response = requests.get(
    "https://api.github.com/search/repositories",
    params={"q": "devops", "sort": "stars", "per_page": 5}
)
repos = response.json()["items"]
for repo in repos:
    print(f"  {repo['full_name']} - Stars: {repo['stargazers_count']}")

# POST request
response = requests.post(
    "https://httpbin.org/post",
    json={"name": "devops-app", "version": "1.0"},
    headers={"Authorization": "Bearer YOUR_TOKEN"}
)
print(response.json())

# Handle errors
response = requests.get("https://api.github.com/users/nonexistent-user-12345")
if response.status_code == 200:
    print("User found!")
elif response.status_code == 404:
    print("User not found!")
else:
    print(f"Error: {response.status_code}")
```

### Health Check Script

```python
import requests
import time

def check_endpoint(url, expected_status=200, timeout=5):
    """Check if an endpoint is healthy."""
    try:
        start = time.time()
        response = requests.get(url, timeout=timeout)
        elapsed = (time.time() - start) * 1000

        status = "OK" if response.status_code == expected_status else "FAIL"
        return {
            "url": url,
            "status": status,
            "code": response.status_code,
            "response_time_ms": round(elapsed, 2)
        }
    except requests.exceptions.Timeout:
        return {"url": url, "status": "TIMEOUT", "code": None, "response_time_ms": None}
    except requests.exceptions.ConnectionError:
        return {"url": url, "status": "UNREACHABLE", "code": None, "response_time_ms": None}

# Check multiple endpoints
endpoints = [
    "https://google.com",
    "https://github.com",
    "https://httpbin.org/status/200",
    "https://httpbin.org/status/500",
]

print(f"{'URL':<40} {'Status':<12} {'Code':<6} {'Time (ms)'}")
print("-" * 75)
for url in endpoints:
    result = check_endpoint(url)
    print(f"{result['url']:<40} {result['status']:<12} {str(result['code']):<6} {result['response_time_ms']}")
```

---

## JSON & YAML Parsing

### JSON

```python
import json

# Python dict to JSON string
config = {
    "app_name": "devops-api",
    "port": 8080,
    "debug": False,
    "databases": {
        "primary": "postgresql://db:5432/app",
        "cache": "redis://cache:6379"
    },
    "allowed_origins": ["https://app.example.com", "https://admin.example.com"]
}

# Write JSON to file
with open("config.json", "w") as f:
    json.dump(config, f, indent=2)

# Read JSON from file
with open("config.json", "r") as f:
    loaded_config = json.load(f)
    print(loaded_config["app_name"])
    print(loaded_config["databases"]["primary"])

# JSON string ↔ Python dict
json_string = '{"name": "web-01", "status": "running"}'
data = json.loads(json_string)       # String → Dict
back_to_string = json.dumps(data, indent=2)  # Dict → String
print(back_to_string)
```

### YAML

```bash
pip install pyyaml
```

```python
import yaml

# Read YAML file
yaml_content = """
app:
  name: devops-api
  port: 8080
  debug: false

databases:
  primary:
    host: db.example.com
    port: 5432
    name: app_db
  cache:
    host: cache.example.com
    port: 6379

services:
  - name: nginx
    port: 80
  - name: api
    port: 3000
  - name: worker
    replicas: 3
"""

# Parse YAML string
config = yaml.safe_load(yaml_content)
print(config["app"]["name"])          # devops-api
print(config["databases"]["primary"]["host"])  # db.example.com

# Iterate services
for service in config["services"]:
    print(f"Service: {service['name']}, Port: {service.get('port', 'N/A')}")

# Write YAML to file
with open("config.yaml", "w") as f:
    yaml.dump(config, f, default_flow_style=False, sort_keys=False)

# Read YAML from file
with open("config.yaml", "r") as f:
    loaded = yaml.safe_load(f)
    print(loaded)
```

---

## Error Handling

```python
import os
import requests

# Basic try-except
try:
    with open("/etc/nonexistent_file.conf", "r") as f:
        content = f.read()
except FileNotFoundError:
    print("Config file not found!")
except PermissionError:
    print("No permission to read the file!")

# Multiple exception types
def safe_api_call(url):
    try:
        response = requests.get(url, timeout=5)
        response.raise_for_status()  # Raises for 4xx/5xx
        return response.json()
    except requests.exceptions.Timeout:
        print(f"Timeout connecting to {url}")
    except requests.exceptions.ConnectionError:
        print(f"Cannot connect to {url}")
    except requests.exceptions.HTTPError as e:
        print(f"HTTP error: {e}")
    except json.JSONDecodeError:
        print("Response is not valid JSON")
    return None

# Try-except-finally
def read_config(filepath):
    f = None
    try:
        f = open(filepath, "r")
        return yaml.safe_load(f)
    except FileNotFoundError:
        print(f"Config not found: {filepath}")
        return {}
    finally:
        if f:
            f.close()
        print("Cleanup complete")

# Custom exceptions
class DeploymentError(Exception):
    pass

class HealthCheckError(Exception):
    def __init__(self, service, status_code):
        self.service = service
        self.status_code = status_code
        super().__init__(f"{service} health check failed with status {status_code}")

# Using custom exceptions
def deploy(service_name):
    try:
        response = requests.get(f"http://{service_name}:8080/health", timeout=3)
        if response.status_code != 200:
            raise HealthCheckError(service_name, response.status_code)
        print(f"{service_name} is healthy!")
    except HealthCheckError as e:
        print(f"Deployment blocked: {e}")
    except requests.exceptions.ConnectionError:
        raise DeploymentError(f"Cannot reach {service_name}")
```

---

## Projects

### Project 1: Weather API Script

```python
#!/usr/bin/env python3
"""Fetch weather data from a public API and display it."""

import requests
import json
from datetime import datetime

def get_weather(city, api_key="demo"):
    """Fetch weather for a city using wttr.in (no API key needed)."""
    url = f"https://wttr.in/{city}?format=j1"
    try:
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"Error fetching weather: {e}")
        return None

def display_weather(data, city):
    """Display weather information."""
    if not data:
        print("No data available.")
        return

    current = data.get("current_condition", [{}])[0]

    print(f"\n{'='*50}")
    print(f"  Weather Report for {city.title()}")
    print(f"  {datetime.now().strftime('%Y-%m-%d %H:%M')}")
    print(f"{'='*50}")
    print(f"  Temperature:  {current.get('temp_C', 'N/A')}°C / {current.get('temp_F', 'N/A')}°F")
    print(f"  Feels Like:   {current.get('FeelsLikeC', 'N/A')}°C")
    print(f"  Humidity:     {current.get('humidity', 'N/A')}%")
    print(f"  Wind:         {current.get('windspeedKmph', 'N/A')} km/h {current.get('winddir16Point', '')}")
    print(f"  Condition:    {current.get('weatherDesc', [{}])[0].get('value', 'N/A')}")
    print(f"  Visibility:   {current.get('visibility', 'N/A')} km")
    print(f"{'='*50}")

    # 3-day forecast
    forecast = data.get("weather", [])
    if forecast:
        print(f"\n  3-Day Forecast:")
        print(f"  {'Date':<12} {'Min°C':<8} {'Max°C':<8} {'Condition'}")
        print(f"  {'-'*45}")
        for day in forecast[:3]:
            condition = day.get("hourly", [{}])[4].get("weatherDesc", [{}])[0].get("value", "N/A")
            print(f"  {day['date']:<12} {day['mintempC']:<8} {day['maxtempC']:<8} {condition}")

    # Save to JSON
    with open(f"weather_{city}.json", "w") as f:
        json.dump(data, f, indent=2)
    print(f"\n  Full data saved to weather_{city}.json")

if __name__ == "__main__":
    cities = ["Mumbai", "London", "Tokyo"]
    for city in cities:
        data = get_weather(city)
        display_weather(data, city)
```

### Project 2: Log File Analyzer

```python
#!/usr/bin/env python3
"""Analyze Apache/Nginx access logs to extract useful information."""

import re
from collections import Counter
from datetime import datetime

# Sample log line:
# 192.168.1.1 - - [15/Jan/2024:10:30:45 +0000] "GET /api/users HTTP/1.1" 200 1234

LOG_PATTERN = re.compile(
    r'(?P<ip>\d+\.\d+\.\d+\.\d+)'
    r' - - '
    r'\[(?P<datetime>[^\]]+)\]'
    r' "(?P<method>\w+) (?P<path>\S+) HTTP/\d\.\d"'
    r' (?P<status>\d+)'
    r' (?P<size>\d+)'
)

def generate_sample_log():
    """Generate a sample log file for testing."""
    import random
    ips = ["192.168.1.1", "192.168.1.2", "10.0.0.5", "172.16.0.10", "203.0.113.50"]
    paths = ["/", "/api/users", "/api/products", "/login", "/static/style.css",
             "/api/orders", "/health", "/admin", "/favicon.ico"]
    statuses = ["200", "200", "200", "200", "301", "404", "500", "403"]
    methods = ["GET", "GET", "GET", "POST", "PUT", "DELETE"]

    lines = []
    for i in range(500):
        ip = random.choice(ips)
        path = random.choice(paths)
        status = random.choice(statuses)
        method = random.choice(methods)
        size = random.randint(100, 50000)
        date = f"15/Jan/2024:{random.randint(0,23):02d}:{random.randint(0,59):02d}:{random.randint(0,59):02d} +0000"
        lines.append(f'{ip} - - [{date}] "{method} {path} HTTP/1.1" {status} {size}')

    with open("access.log", "w") as f:
        f.write("\n".join(lines))
    print(f"Generated sample log with {len(lines)} entries.")

def analyze_log(filepath):
    """Analyze an access log file."""
    ip_counter = Counter()
    path_counter = Counter()
    status_counter = Counter()
    method_counter = Counter()
    total_bytes = 0
    error_requests = []
    total_lines = 0
    parsed_lines = 0

    with open(filepath, "r") as f:
        for line in f:
            total_lines += 1
            match = LOG_PATTERN.match(line.strip())
            if not match:
                continue

            parsed_lines += 1
            data = match.groupdict()

            ip_counter[data["ip"]] += 1
            path_counter[data["path"]] += 1
            status_counter[data["status"]] += 1
            method_counter[data["method"]] += 1
            total_bytes += int(data["size"])

            if int(data["status"]) >= 400:
                error_requests.append(data)

    # Print report
    print(f"\n{'='*60}")
    print(f"  ACCESS LOG ANALYSIS REPORT")
    print(f"  File: {filepath}")
    print(f"  Total lines: {total_lines} | Parsed: {parsed_lines}")
    print(f"{'='*60}")

    print(f"\n  TOP 10 IP ADDRESSES:")
    print(f"  {'IP Address':<20} {'Requests':<10} {'Percentage'}")
    print(f"  {'-'*45}")
    for ip, count in ip_counter.most_common(10):
        pct = (count / parsed_lines) * 100
        print(f"  {ip:<20} {count:<10} {pct:.1f}%")

    print(f"\n  TOP 10 REQUESTED PATHS:")
    print(f"  {'Path':<30} {'Requests':<10}")
    print(f"  {'-'*40}")
    for path, count in path_counter.most_common(10):
        print(f"  {path:<30} {count}")

    print(f"\n  STATUS CODE DISTRIBUTION:")
    print(f"  {'Status':<10} {'Count':<10} {'Description'}")
    print(f"  {'-'*40}")
    status_desc = {"200": "OK", "301": "Redirect", "404": "Not Found", "403": "Forbidden", "500": "Server Error"}
    for status, count in sorted(status_counter.items()):
        desc = status_desc.get(status, "Other")
        print(f"  {status:<10} {count:<10} {desc}")

    print(f"\n  HTTP METHODS:")
    for method, count in method_counter.most_common():
        print(f"  {method:<10} {count}")

    print(f"\n  SUMMARY:")
    print(f"  Total data transferred: {total_bytes / 1024 / 1024:.2f} MB")
    print(f"  Error requests (4xx/5xx): {len(error_requests)}")
    print(f"  Error rate: {(len(error_requests) / parsed_lines * 100):.1f}%")
    print(f"{'='*60}\n")

if __name__ == "__main__":
    generate_sample_log()
    analyze_log("access.log")
```

### Project 3: Shell Command Orchestrator

```python
#!/usr/bin/env python3
"""Orchestrate a series of shell commands with status reporting."""

import subprocess
import sys
import time
from datetime import datetime

class CommandRunner:
    def __init__(self, name):
        self.name = name
        self.results = []
        self.start_time = None

    def run(self, description, command, shell=False, allow_fail=False):
        """Run a command and track its result."""
        print(f"\n  [{len(self.results)+1}] {description}")
        print(f"      Command: {command if isinstance(command, str) else ' '.join(command)}")

        start = time.time()
        try:
            result = subprocess.run(
                command,
                shell=shell,
                capture_output=True,
                text=True,
                timeout=60
            )
            elapsed = time.time() - start

            success = result.returncode == 0
            status = "PASS" if success else "FAIL"

            self.results.append({
                "description": description,
                "command": command,
                "status": status,
                "returncode": result.returncode,
                "stdout": result.stdout,
                "stderr": result.stderr,
                "elapsed": elapsed
            })

            if success:
                print(f"      Status: PASS ({elapsed:.2f}s)")
                if result.stdout.strip():
                    for line in result.stdout.strip().split("\n")[:5]:
                        print(f"      > {line}")
            else:
                print(f"      Status: FAIL (exit code: {result.returncode})")
                if result.stderr.strip():
                    print(f"      Error: {result.stderr.strip()[:200]}")
                if not allow_fail:
                    print(f"\n  PIPELINE STOPPED: Command failed and allow_fail=False")
                    return False

            return True

        except subprocess.TimeoutExpired:
            elapsed = time.time() - start
            self.results.append({
                "description": description,
                "command": command,
                "status": "TIMEOUT",
                "returncode": -1,
                "stdout": "",
                "stderr": "Command timed out after 60 seconds",
                "elapsed": elapsed
            })
            print(f"      Status: TIMEOUT ({elapsed:.2f}s)")
            return allow_fail

    def report(self):
        """Print a summary report of all commands."""
        passed = sum(1 for r in self.results if r["status"] == "PASS")
        failed = sum(1 for r in self.results if r["status"] == "FAIL")
        timeouts = sum(1 for r in self.results if r["status"] == "TIMEOUT")
        total_time = sum(r["elapsed"] for r in self.results)

        print(f"\n{'='*60}")
        print(f"  PIPELINE REPORT: {self.name}")
        print(f"  {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        print(f"{'='*60}")
        print(f"  {'#':<4} {'Status':<10} {'Time':<10} {'Description'}")
        print(f"  {'-'*55}")

        for i, r in enumerate(self.results, 1):
            print(f"  {i:<4} {r['status']:<10} {r['elapsed']:.2f}s     {r['description']}")

        print(f"  {'-'*55}")
        print(f"  Total:  {len(self.results)} commands | "
              f"Passed: {passed} | Failed: {failed} | Timeouts: {timeouts}")
        print(f"  Duration: {total_time:.2f}s")
        print(f"  Result: {'SUCCESS' if failed == 0 and timeouts == 0 else 'FAILURE'}")
        print(f"{'='*60}\n")

        return failed == 0 and timeouts == 0

# Example: Server setup pipeline
if __name__ == "__main__":
    runner = CommandRunner("Server Health Check Pipeline")

    print(f"\n{'='*60}")
    print(f"  Starting: {runner.name}")
    print(f"{'='*60}")

    # Run health checks
    runner.run("Check disk space", "df -h", shell=True)
    runner.run("Check memory usage", "free -h", shell=True)
    runner.run("Check running processes", "ps aux | head -15", shell=True)
    runner.run("Check network connectivity", ["ping", "-c", "2", "google.com"])
    runner.run("Check DNS resolution", ["nslookup", "github.com"], allow_fail=True)
    runner.run("List open ports", "ss -tlnp", shell=True, allow_fail=True)
    runner.run("Check system uptime", ["uptime"])

    success = runner.report()
    sys.exit(0 if success else 1)
```

```bash
# Run the scripts
chmod +x *.py
python3 weather_api.py
python3 log_analyzer.py
python3 command_runner.py
```
