
# 🐱 Backend Wizards — Stage 0 Task: Build a Dynamic Profile Endpoint

This project implements a simple **RESTful API** that returns dynamic profile information along with a random cat fact fetched from the [Cat Facts API](https://catfact.ninja/fact).
It’s built with **Python 3.12 + Flask** and hosted on **AWS EC2 (Ubuntu 22.04)**.

---

## 🚀 Features

* `GET /me` endpoint that returns:

  * Your **name**, **email**, and **backend stack**
  * A **dynamic UTC timestamp**
  * A **random cat fact** fetched from Cat Facts API
* Graceful handling of API or network errors
* JSON response formatted exactly as required by the Stage 0 specification

---

## 📂 Project Structure

```
backend-stage0/
├── app.py
├── requirements.txt
├── README.md
└── venv/  (optional virtual environment folder)
```

---

## ⚙️ Setup Instructions

### 1️⃣ Clone the Repository

```bash
git clone https://github.com/<your-username>/backend-stage0.git
cd backend-stage0
```

### 2️⃣ Create and Activate Virtual Environment

```bash
python3 -m venv venv
source venv/bin/activate
```

### 3️⃣ Install Dependencies

```bash
pip install -r requirements.txt
```

*(If `requirements.txt` doesn’t exist, install directly and then create it:)*

```bash
pip install flask requests
pip freeze > requirements.txt
```

---

## ▶️ Run Locally

```bash
python app.py
```

Your local server will start at:

```
http://127.0.0.1:5000/me
```

Test using your browser or a tool like **Postman**:

```
GET http://127.0.0.1:5000/me
```

---

## 🌍 Deployed Version

Live endpoint:
👉 **http://<your-ec2-public-ip>/me**

*(Replace `<your-ec2-public-ip>` with your EC2 instance public IP address)*

---

## 🧩 Environment Variables

*(Optional — none required by default)*
If you want to use environment variables instead of hardcoding your details:

```bash
export USER_NAME="John Abioye"
export USER_EMAIL="ja-abioye@example.com"
export USER_STACK="Python/Flask"
```

---

## 🧪 Example Response

```json
{
  "status": "success",
  "user": {
    "email": "ja-abioye@example.com",
    "name": "John Abioye",
    "stack": "Python/Flask"
  },
  "timestamp": "2025-10-18T21:30:56.789Z",
  "fact": "Cats can rotate their ears 180 degrees."
}
```

---

## 🧰 Dependencies

| Package      | Purpose                                |
| ------------ | -------------------------------------- |
| **Flask**    | Framework for building the RESTful API |
| **Requests** | For consuming the Cat Facts API        |

---

## 📘 API Documentation

| Method  | Endpoint | Description                              | Response Code | Example                                                                     |
| ------- | -------- | ---------------------------------------- | ------------- | --------------------------------------------------------------------------- |
| **GET** | `/me`    | Returns profile info + a random cat fact | 200 OK        | `{ "status": "success", "user": {...}, "timestamp": "...", "fact": "..." }` |

### ✅ Example Request

```bash
curl http://127.0.0.1:5000/me
```

### ✅ Example Successful Response

```json
{
  "status": "success",
  "user": {
    "email": "ja-abioye@example.com",
    "name": "John Abioye",
    "stack": "Python/Flask"
  },
  "timestamp": "2025-10-18T21:30:56.789Z",
  "fact": "A group of cats is called a clowder."
}
```

### ⚠️ Example Error Response (if Cat Facts API is unreachable)

```json
{
  "status": "error",
  "message": "Unable to fetch cat fact at the moment. Please try again later."
}
```

---

## 🧠 Testing the Endpoint

Test locally:

```bash
curl http://127.0.0.1:5000/me
```

Or test the live EC2 deployment:

```bash
curl http://<your-ec2-public-ip>/me
```

---

## 📝 Notes

* Ensure **port 5000** is open and proxied through Nginx.
* Restart Flask after any code updates.
* Always activate your venv before running the app.

---

## 👨‍💻 Author

**John Abioye (@John Billon)**
Backend Developer | AWS Certified | Cloud & DevOps Engineer

📧 [adeolujohn495@gmail.com](mailto:adeolujohn495@gmail.com)
🐙 GitHub: [github.com/BILLIYON]([(https://github.com/BILLIYON/HNG-Backend-Wizards-Stage-0-Task)])
🌐 Deployed API: [http://54.158.6.245/me](http://54.158.6.245/me)

---

Would you like me to include a short **license section (MIT)** and a **“What I Learned” reflection paragraph** at the end to make it look more polished for reviewers and portfolio readers?
