# Coherent step-by-step writeup — what we did, why, and how it works

Below is a single, coherent run-through of every major step I completed, what each file/folder does, why I did it, and how it all fits together.
---

## 1) Project goal (high level)

Build two Stage-0 tasks on one EC2 instance:

* **DevOps Stage 0:** Serve a static HTML landing page via **Nginx** on port **80**.
* **Backend Stage 0:** Provide a **dynamic `/me` API** (returns profile + cat fact) using **Python/Flask**, proxied by Nginx so the API is reachable at `http://<EC2-IP>/me`.

Why: demonstrates provisioning, service configuration, reverse proxying, third-party API consumption, and deployment best practices.

---

## 2) Instance provisioning & initial setup

### What you did

* Launched an **AWS EC2** instance (Ubuntu 22.04).
* Generated/used an SSH key pair to access the server (`.pem` on the local machine).
* Optionally used **user-data** to bootstrap Nginx and drop the initial `index.html`.

### Why

* EC2 provides a public VM to host both static and dynamic content.
* SSH keys provide secure access without passwords.
* User-data automates initial setup so the instance serves the page immediately after boot.

### How it works

* EC2 user-data runs once at first boot and can run shell commands that install packages and create files.
* SSH connects you to the instance so you can manage files and services.

---

## 3) Nginx static site (DevOps Stage 0)

### Files & commands

* `/var/www/html/index.html` — your static landing page with: `Welcome to DevOps Stage 0 - [Your Name]/[Slack]`
* Commands used:

  * `sudo apt update && sudo apt install nginx -y`
  * `sudo systemctl enable nginx && sudo systemctl start nginx`
  * Edit file: `sudo nano /var/www/html/index.html`
  * Permissions: `sudo chown www-data:www-data /var/www/html/index.html && sudo chmod 644 ...`

### Why

* Nginx is a production-grade, lightweight web server ideal for serving static files and reverse proxying to application servers.
* Serving the static page on port **80** meets the task requirement (only port 80 allowed for grading).

### How it works

* Nginx listens on port 80 and serves `index.html` for `/` requests.
* Ownership and permissions ensure the `www-data` user serves the file safely.

---

## 4) Networking: Security Group, NACL, and troubleshooting

### What happened

* You had `curl` working on the instance but the public IP initially refused connection.
* Found Security Group allowed port 80 but NACL had a Deny rule preventing traffic.
* Fixed by using a custom NACL (or editing the NACL rules) and ensuring SG inbound allowed HTTP (0.0.0.0/0) and SSH (from your IP).

### Why

* AWS security is layered: **Security Group (stateful)** and **Network ACL (stateless)** both influence connectivity. NACLs are evaluated by rule number and can block traffic even when SG allows it.
* Ensuring proper rules avoids false negatives when testing from external networks.

### How it works

* Security Group allows return traffic automatically; NACL requires explicit return rules for ephemeral ports.
* After NACL fixed, external `curl http://<IP>` returned your page.

---

## 5) Preparing the Flask app (Backend Stage 0)

### Project files

* `app.py` — main Flask application implementing `GET /me`.
* `requirements.txt` — `flask` and `requests`.
* `README.md` — instructions, API docs, example responses.
* `venv/` — virtual environment (should be ignored in Git).

### Key `app.py` behavior

* Endpoint: `GET /me`
* Builds response:

  ```json
  {
    "status":"success",
    "user": { "email": "...", "name": "...", "stack":"..." },
    "timestamp": "<current UTC ISO 8601>",
    "fact": "<random-cat-fact or fallback>"
  }
  ```
* Fetches cat fact from `https://catfact.ninja/fact` with timeout & error handling.

### Why

* Flask is lightweight, easy to set up and ideal for small REST endpoints.
* Using a separate app process (on 5000) isolates the dynamic API from Nginx’s static file handling.

### How it works

* Flask listens on the local loopback (`127.0.0.1`) or `0.0.0.0` on a non-privileged port (5000).
* Nginx can forward requests to it using a reverse proxy configuration.

---

## 6) Virtual environment & dependencies

### Steps

* Install `python3-venv` (match the OS Python version).
* Create venv: `python3 -m venv venv`
* Activate: `source venv/bin/activate`
* Install packages: `pip install flask requests`
* Freeze: `pip freeze > requirements.txt`

### Why

* Virtualenv isolates project dependencies from system Python and avoids permission issues (PEP 668 errors).
* Ensures reproducible installs for reviewers and CI.

### How it works

* `venv` creates a folder with a private Python interpreter and pip.
* Activating alters PATH so `pip`/`python` refer to the venv versions.

---

## 7) Nginx reverse proxy configuration

### Key Nginx config snippet (in `/etc/nginx/sites-available/default`):

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /me {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Why

* Nginx can serve static files and forward API calls seamlessly, letting both services run on one machine and on the standard HTTP port 80 required by the challenge.

### How it works

* Requests to `/me` hit Nginx → Nginx forwards (proxy_pass) to Flask listening on localhost:5000 → Flask returns JSON → Nginx relays response to client.
* Proxy headers preserve client IP and scheme for logging/troubleshooting.

---

## 8) Running Flask persistently

### Short-term

* `nohup python3 app.py &` to background the process.

### Better long-term (recommended)

* Create a **systemd** service unit `/etc/systemd/system/backend-stage0.service`:

```ini
[Unit]
Description=Flask backend-stage0
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/backend-stage0
Environment="PATH=/home/ubuntu/backend-stage0/venv/bin"
ExecStart=/home/ubuntu/backend-stage0/venv/bin/python /home/ubuntu/backend-stage0/app.py
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

* Commands:

  ```bash
  sudo systemctl daemon-reload
  sudo systemctl enable backend-stage0
  sudo systemctl start backend-stage0
  sudo systemctl status backend-stage0
  ```

### Why

* `nohup` is quick but fragile (won’t auto-restart on reboot). `systemd` is robust, auto-starts on boot, and restarts the service on crash.

### How it works

* systemd launches the service using the venv interpreter, keeps it running, and allows standard logging via `journalctl -u backend-stage0`.

---

## 9) Testing & verification

### Commands used

* Local instance test: `curl http://localhost:5000/me`
* From outside: `curl http://<public-ip>/me`
* Nginx config test: `sudo nginx -t`
* Service status: `sudo systemctl status nginx` and `sudo systemctl status backend-stage0`
* Port listening: `sudo ss -tuln | grep 5000` or `sudo lsof -i :5000`

### Why

* Confirm each layer works: app process, reverse proxy, network rules, and external reachability.

---

## 10) Troubleshooting highlights you encountered

* **“Refused to connect”**: initially caused by NACL deny rule — fixed by creating/editing a NACL or reassigning subnet’s NACL and ensuring SG allowed HTTP.
* **Flask module not found / PEP 668**: solved by installing `python3-venv` and using a virtual environment rather than installing system-wide packages.
* **Nginx config errors**: resolved by ensuring directives are inside a single `server { ... }` block and running `nginx -t` to test before reload.
* **Daemon reload warning**: run `sudo systemctl daemon-reload` before restarting when system files changed.

---

## 11) GitHub repo & repo files

* **What to push**: `app.py`, `requirements.txt`, `README.md`, `.gitignore` (include `venv/`).
* **What *not* to push**: `venv/`, `.pem` keys, any secrets or private data.
* README should include install steps, run instructions, example responses, and the live URL: `http://<ec2-ip>/me`.

---

## 12) Submission checklist (for HNG bot)

* [x] `GET /me` returns **200 OK**.
* [x] Response JSON exactly matches required fields (`status`, `user.email`, `user.name`, `user.stack`, `timestamp` in ISO8601, `fact`).
* [x] Response `Content-Type: application/json`.
* [x] Live URL is reachable from multiple networks (`http://<ec2-ip>/me`).
* [x] GitHub repo link with README and instructions provided in Slack bot `/stage-zero-backend`.
* [x] Keep the server running until grading completes.

---

## 13) Next steps & best practices (recommended)

* Add **logging** to `app.py` (use `logging` module) for easier debugging.
* Add **unit tests** (e.g., pytest) and an example integration test that mocks the Cat Facts API.
* Use environment variables for user info (so you don’t hardcode personal data).
* Consider using **gunicorn** behind Nginx instead of running Flask’s dev server in production:

  ```bash
  pip install gunicorn
  gunicorn -w 3 -b 127.0.0.1:5000 app:app
  ```

  Update systemd ExecStart accordingly.
* Secure the instance: rotate SSH keys, disable password auth, and restrict SSH to your IP.
* After grading, **terminate** or **stop** the EC2 instance to avoid charges.

---

## 14) Quick reference commands (copy/paste)

```bash
# site files
sudo nano /var/www/html/index.html

# go to project
cd ~/backend-stage0

# create/activate venv
python3 -m venv venv
source venv/bin/activate

# install deps
pip install flask requests
pip freeze > requirements.txt

# start app (dev)
python3 app.py

# start app (background)
nohup python3 app.py &

# test locally
curl http://127.0.0.1:5000/me

# test externally
curl http://<EC2-PUBLIC-IP>/me

# nginx test and reload
sudo nginx -t
sudo systemctl reload nginx

# create systemd service (after adding file)
sudo systemctl daemon-reload
sudo systemctl enable backend-stage0
sudo systemctl start backend-stage0
sudo systemctl status backend-stage0
```

---

