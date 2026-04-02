# A-LMS — Agentic Learning Management System

> *A full-stack AI-powered platform where five specialized agents collaborate to grade student work, generate personalized feedback, and adapt learning paths — with teachers always in the loop.*

---

 an AI-powered Learning Management System that I built milestone by milestone. The core idea is simple: grading subjective work is exhausting and time-consuming for teachers, and generic feedback doesn't really help students grow. A-LMS tries to fix both of those problems using a pipeline of AI agents that evaluate student submissions against a rubric, generate detailed feedback, and then let the teacher review everything before the student ever sees it.

Think of it as AI doing the heavy lifting, with teachers keeping full control.

---

## What can it actually do?

### For Teachers
- **Create and manage classrooms** — each classroom is private, invite-code based, and completely isolated from other teachers' classes (multi-tenancy built in)
- **Build any type of coursework** — quizzes, essays, assignments, or case studies
- **Upload rubrics** — paste a .txt or .pdf rubric file and let the AI parse it into structured criteria automatically
- **Review AI grades before students see them** — the system flags every AI-graded submission as "Pending Review" so you can edit, add your own notes, and approve
- **See class-wide analytics** — understand which concepts your whole class is struggling with at a glance

### For Students
- **Join classrooms** via invite code from their teacher
- **Take quizzes** and get instant scores
- **Submit essays and case studies** and see exactly how they were graded, criterion by criterion
- **Get a personalized learning path** — the system tracks what each student knows and generates targeted practice for their weak spots

---

## The 5 AI Agents (the fun part)

This is where things get interesting. Under the hood, student submissions pass through a pipeline of specialized agents:

**1. Quiz Auto-Grader**
The simplest one — it checks quiz answers against the teacher's answer key and scores them instantly. No AI needed here, just logic.

**2. Requirement Agent**
For essays and assignments, this agent checks the basics: Is it long enough? Does it have the right structure? It's the first gate in the evaluation chain.

**3. Rubric Alignment Agent**
This is the main workhorse. It takes the student's submission and the teacher's rubric, sends both to Gemini 1.5 Flash, and gets back a score and justification for every single criterion. The output is structured — not a blob of text, but a proper JSON object with scores and reasoning.

**4. Synthesis Agent**
Takes everything from agents 2 and 3 and combines it into one coherent, readable feedback report for the student.

**5. Student Model Agent + Pedagogical Planner**
After the teacher approves a grade, these two work together. The Student Model Agent updates each student's "Dynamic Knowledge Graph" — a persistent record of what they know and don't know. The Pedagogical Planner then reads that graph and decides what remedial exercises the student should do next.

The agents are built with **LangGraph** and run as a **Celery** background task so the API stays fast and responsive while grading happens asynchronously.

---

## Tech Stack

| Layer | What's used |
|---|---|
| Backend | FastAPI, SQLAlchemy, SQLite (dev) / PostgreSQL (prod) |
| AI Agents | LangGraph, LangChain, Gemini 1.5 Flash |
| Task Queue | Celery + Redis |
| Auth | JWT (python-jose), bcrypt (passlib) |
| Frontend | React 18, React Router v6, Axios |
| PDF Parsing | pypdf |

---

## Getting started

You'll need **4 terminal windows** running simultaneously. Yes, four. That's the price of having a real task queue.

### 1. Clone and set up

```bash
git clone https://github.com/your-username/AdaptiveAgentLoop.git
cd AdaptiveAgentLoop
```

### 2. Backend setup

```bash
cd backend
python -m venv venv

# Windows
.\venv\Scripts\activate

# Mac / Linux
source venv/bin/activate

pip install fastapi "uvicorn[standard]" sqlalchemy passlib[bcrypt] \
  "python-jose[cryptography]" langchain langchain-google-genai \
  langgraph celery redis pypdf
```

Create a `.env` file in the `backend/` folder:

```
GOOGLE_API_KEY=your_gemini_api_key_here
SECRET_KEY=your_jwt_secret_here
```

### 3. Frontend setup

```bash
cd frontend
npm install
```

### 4. Run everything

Open 4 terminals and run one command in each:

```bash
# Terminal 1 — Redis (must be installed separately)
redis-server

# Terminal 2 — FastAPI backend
cd backend
uvicorn app.main:app --reload

# Terminal 3 — Celery worker (handles AI grading in background)
cd backend
celery -A app.celery_worker.celery_app worker --loglevel=info -P solo

# Terminal 4 — React frontend
cd frontend
npm start
```

Then open `http://localhost:3000` in your browser.

> **Important:** On first run, you need to manually approve a teacher account in the database. Open `backend/sql_app.db` with any SQLite viewer, find your teacher user, and set `is_approved = 1`.

---

## Trying it out (the full flow)

Here's the complete happy path to test everything end to end:

1. **Register a Teacher account** → get it approved in the DB → log in
2. **Create a classroom** → note the 6-character invite code
3. **Create an Essay coursework** → use the rubric builder or upload a .txt rubric → let AI parse it
4. **Register a Student account** → log in as the student → join the classroom with the invite code
5. **Open the essay and submit something** — watch the status change to `GRADING`, then `PENDING_REVIEW`
6. **Switch back to the teacher account** → open the Submissions dashboard → review the AI's feedback → edit if needed → click Approve
7. **Back as the student** → the submission now shows `GRADED` with the full rubric breakdown and your teacher's notes

---

## Project milestones

This was built incrementally over several milestones:

| Milestone | What was built |
|---|---|
| **M0** | Backend + frontend "Hello World" handshake, Git setup, CORS |
| **M1** | JWT authentication, role-based access (teacher/student), SQLAlchemy User model |
| **M2** | Classrooms, invite codes, student enrollment, multi-tenancy |
| **M3** | Quiz creation, student submissions, Quiz Auto-Grader background agent |
| **M4** | Essay/case study flow, 3-agent evaluation chain, rubric upload + AI parsing, deadline enforcement, single-attempt checks, detailed submission results |
| **M5 (upcoming)** | Dynamic Student Knowledge Graph, Student Model Agent, Pedagogical Planner, class analytics dashboard |

---

## A note on the "Human-in-the-Loop" design

One thing I was deliberate about: the AI never talks directly to students. Every AI-generated grade goes through a `PENDING_REVIEW` status where the teacher can see exactly what the AI said, disagree with it, edit scores, add their own comments, and only then release it.

This matters because:
- AI can miss context that a teacher would catch (sarcasm, creative risk-taking, personal circumstances)
- Teachers should understand what feedback their students are receiving
- Students deserve to know that a human reviewed their work, not just an algorithm

The whole "Human-in-the-Loop" (HoTL) pattern is baked into the submission status lifecycle: `SUBMITTED → GRADING → PENDING_REVIEW → GRADED`.

---

## Known issues / things to fix

- The teacher approval flow is manual (requires direct DB edit) — a proper admin panel is on the to-do list
- The `DSKG` (Dynamic Student Knowledge Graph) and personalized learning path are designed but not yet implemented (coming in M5)
- SQLite works fine for development but should be swapped for PostgreSQL before any real deployment
- There's no file upload for student submissions yet — only text. PDF/DOCX uploads are planned

---

