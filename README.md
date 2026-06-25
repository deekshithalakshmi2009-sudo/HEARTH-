<div align="center">
  
# 🔥 HEARTH
### Predict. Prevent. Protect.

*A dual-purpose civic platform for homelessness prevention and city infrastructure management.*

**🌍 Live Demo: [https://hearth-city.netlify.app](https://hearth-city.netlify.app)**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![React](https://img.shields.io/badge/React-20232A?style=flat&logo=react&logoColor=61DAFB)](https://reactjs.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![TailwindCSS](https://img.shields.io/badge/Tailwind_CSS-38B2AC?style=flat&logo=tailwind-css&logoColor=white)](https://tailwindcss.com/)

[Features](#-key-features) • [Tech Stack](#-tech-stack) • [Getting Started](#-getting-started)

</div>

---

## 🌟 The Vision

City governments struggle to manage the sheer volume of civic issues—from critical homelessness crises to infrastructure failures like broken streetlights and potholes. **HEARTH** bridges the gap between citizens and city officials, providing a unified, real-time dashboard to report, track, and resolve problems faster than ever.

By empowering citizens to report issues with photographic evidence and giving officials a bird's-eye view of the entire city, Hearth transforms reactive governance into **proactive public service.**

---

## ✨ Key Features

| Feature | Description | Impact |
| :--- | :--- | :--- |
| **🏠 Housing Crisis Hub** | Report eviction risks, encampments, and mental health crises. | Rapid outreach dispatching. |
| **🛣️ Infra Reporting** | Report potholes, broken lights, and water leaks with **Photo Uploads**. | Actionable evidence for work crews. |
| **🛰️ Live City Map** | Interactive geospatial mapping of all reported incidents. | Visualizing problem hotspots instantly. |
| **📦 Status Tracker** | Amazon-style progress tracker for citizens (`Reported` ➔ `In Progress`). | Unprecedented transparency. |
| **😊 Satisfaction Surveys** | Citizens rate the resolution of their issues. | Holds city departments accountable. |
| **🚨 Auto-Reraise Logic** | If unsatisfied, citizens can instantly re-open reports. | Ensures no issue slips through the cracks. |
| **📊 Mayor's Dashboard** | High-level analytics on resolution times and taxpayer savings. | Data-driven resource allocation. |
| **📄 PDF Engine** | One-click export of complete, filtered city reports. | Beautiful summaries for city council. |

---

## 🛠️ Tech Stack

### Frontend
* **React.js** (Vite) - Lightning-fast UI rendering
* **Tailwind CSS** - Glassmorphism, ultra-modern styling
* **Framer Motion** - Silky smooth micro-animations
* **Leaflet / React-Leaflet** - Geospatial mapping
* **jsPDF / html2canvas** - Client-side PDF generation

### Backend
* **Python / FastAPI** - High-performance asynchronous API
* **WebSockets** - Real-time map updates
* **Pydantic** - Strict data validation

---

## 🚀 Getting Started

Follow these steps to run HEARTH locally on your machine.

### Prerequisites
* [Node.js](https://nodejs.org/) (v16+)
* [Python](https://python.org/) (3.9+)

### 1. Clone the Repository
```bash
git clone https://github.com/hemanthhariharanmaddila-0809/HEARTH-.git
cd HEARTH-
```

### 2. Start the Backend (FastAPI)
```bash
cd backend
# Create and activate a virtual environment (optional but recommended)
python -m venv venv
# Windows: venv\Scripts\activate | Mac/Linux: source venv/bin/activate

# Install dependencies
pip install fastapi uvicorn pydantic

# Run the server
python -m uvicorn main:app --reload
```
*The backend will be running at `http://localhost:8000`*

### 3. Start the Frontend (React)
Open a new terminal window:
```bash
cd frontend

# Install dependencies
npm install

# Start the dev server
npm run dev
```
*The frontend will be running at `http://localhost:5173`*

---

## 👥 Demo Credentials

To access the **Officer Portal**, use the following dummy credentials:

| Role | Username | Password |
| :--- | :--- | :--- |
| **Mayor** | `mayor.sf` | `hearth2024` |
| **Director** | `director.hs`| `hearth2024` |
| **Outreach Officer**| `officer1` | `hearth2024` |

*Citizens do not need a password. They just enter their Government ID (e.g., `IND123456`) to file reports and track statuses.*

---

## 📜 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

<div align="center">
  <br>
  <i>Built with ❤️ for FutureHacks 2026. Making our cities smarter, safer, and kinder.</i>
</div>
