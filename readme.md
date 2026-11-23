# Nutrient Meal Planner

A Django web application that calculates a user's daily calorie and macronutrient targets from their body profile and goal (lose, maintain or gain weight), generates a personalised meal plan, and tracks what they actually eat against those targets.

> **Course:** Software Design and Architecture (SDA) <br> 
> **Focus of this project:** design quality, not feature count. The feature set is deliberately small so the effort goes into architecture, design patterns, SOLID and GRASP principles, and non-functional requirements.

**Team:** _Abdul Hadi (lead), Member 1, Member 2, Member 3_

**Stack:** Python, Django, Django templates, SQLite (dev), PostgreSQL (final demo)

---

## 1. Project Goals

1. Show a clean **layered architecture** where each layer only calls the layer below it.
2. Apply **design patterns** where they solve a real problem (Strategy, Factory, Repository, Observer).
3. Demonstrate **SOLID** and **GRASP** with concrete classes in the code, not just in the report.
4. Meet measurable **non-functional requirements** (performance, accuracy, security, usability, maintainability, testability) and prove them with tests.
5. Deliver a working product: authentication, personalised plans, macro tracking.

---

## 2. Functional Requirements

### Actors
- **User:** manages their profile, generates plans, logs meals, views progress.
- **Admin:** manages the food database.

### Requirements

| ID | Requirement | Module |
|----|-------------|--------|
| FR-1 | A visitor can register with email and password | accounts |
| FR-2 | A registered user can log in and log out | accounts |
| FR-3 | A user can reset their password | accounts |
| FR-4 | A user can enter and edit a profile: age, sex, height, weight, activity level, goal (lose / maintain / gain), target rate of change, dietary preference (e.g. vegetarian) | accounts |
| FR-5 | The system calculates BMR and TDEE from the profile | nutrition |
| FR-6 | The system adjusts calories for the goal and splits them into protein, carbs and fat targets | nutrition |
| FR-7 | The system stores a food database (50 to 100 seeded items) with calories, protein, carbs and fat per serving | foods |
| FR-8 | Admin can add, edit and delete foods | foods |
| FR-9 | The user can generate a 1-day or 7-day meal plan that meets their targets and respects their dietary preference | planner |
| FR-10 | The user can choose a planning style (balanced, high-protein, vegetarian) | planner |
| FR-11 | The user can log foods eaten each day | tracking |
| FR-12 | The user sees daily progress for calories and each macro against targets | tracking |
| FR-13 | The system alerts the user when the day's log goes over or under target | tracking |
| FR-14 | The user can request an AI-written explanation and tips for their plan (optional, can fail without breaking the app) | planner / ai |

### Out of scope (future work)
User-to-user sharing, recipe uploads, barcode scanning, shopping lists, payments, mobile apps.

---

## 3. Authentication and Authorisation

- Built on Django's `django.contrib.auth` with a custom `Profile` model linked one-to-one to `User`.
- Passwords hashed by Django (PBKDF2 by default). No plain-text storage anywhere.
- Session-based login. CSRF protection stays on for all forms.
- **Ownership checks:** every profile, plan and log query is filtered by the logged-in user. A user can never read another user's health data (NFR Security).
- Admin access uses Django's `is_staff` flag and is limited to the food database.
- Ownership checks live in the service/repository layer, not scattered across views.

---

## 4. Architecture

Layered architecture. Each layer calls only the one directly below it.

```
┌──────────────────────────────────────────────┐
│ 1. Presentation   views, forms, templates    │
├──────────────────────────────────────────────┤
│ 2. Application    services (one per use case)│
├──────────────────────────────────────────────┤
│ 3. Domain         models, business rules,    │
│                   strategies, calculators    │
├──────────────────────────────────────────────┤
│ 4. Data access    repositories (all queries) │
├──────────────────────────────────────────────┤
│ 5. Database       SQLite (dev) / PostgreSQL  │
└──────────────────────────────────────────────┘
        External: FoodDataProvider, LLMProvider (behind interfaces)
```

**Rule:** views never touch models or queries directly. They call a service. Services never write SQL. They call a repository.

### Django apps

| App | Responsibility |
|-----|----------------|
| `accounts` | Registration, login, password reset, profile |
| `nutrition` | BMR, TDEE and macro target calculation |
| `foods` | Food items, nutrient data, food repository |
| `planner` | Meal plan generation, diet strategies, AI explanations |
| `tracking` | Daily food log, progress, alerts |

---

## 5. Personalisation Logic (weight loss / gain)

Calculation is **deterministic and formula-based**. It is not done by an LLM.

1. **BMR** using Mifflin-St Jeor (default) or Harris-Benedict, both implemented as interchangeable strategies.
2. **TDEE** = BMR x activity multiplier.
3. **Goal adjustment:**
   - Lose: TDEE minus a daily deficit
   - Maintain: TDEE
   - Gain: TDEE plus a daily surplus
4. **Macro split** by goal and plan style (e.g. higher protein per kg of body weight when losing weight).
5. **Safety limits** enforced in the domain layer: a minimum daily calorie floor, and a cap on the deficit or surplus so the system never recommends extreme targets. The app shows a "not medical advice" notice.

Why formulas and not an LLM: the course requires **accuracy** (results must match published formulas) and **testability** (known inputs give known outputs). An LLM cannot guarantee either.

---

## 6. AI / Dataset Integration

### Where the data comes from

| Need | Choice | Why |
|------|--------|-----|
| Nutrient data for foods | **USDA FoodData Central** (free API key) to build the seed set, plus a few manually entered local foods | Reliable, free, well documented |
| Alternative / backup | **Open Food Facts** (free, open data) | No cost, no key needed for reads |
| Seed strategy | Export 50 to 100 items once into a fixture file (`foods/fixtures/foods.json`) and load into the database | No dependency on an external API at runtime, which also helps the 3-second performance target |

### Where an LLM is used (and where it isn't)

The LLM is used **only** for FR-14: turning a finished plan into a friendly explanation, swap suggestions and tips. It never computes calories or macros and never decides what is safe.

**Recommended free option:** Google Gemini API through a free Google AI Studio key (Flash-class models).
- Pros: free tier with no card, enough for a student demo.
- Cons: free limits are small and change often. Check the live limits in AI Studio before the demo and treat any number you read online as a snapshot.

**Backup options:** a local model through Ollama (fully free and offline, needs a decent laptop), or another provider with a free tier.

### Design rule: the LLM sits behind an interface

```python
class LLMProvider(ABC):
    def explain_plan(self, plan, targets) -> str: ...
```

- `GeminiProvider`, `OllamaProvider` and `NoAIProvider` all implement it.
- Services depend on `LLMProvider`, not on a vendor (**Dependency Inversion**).
- Switching vendors needs no change to existing code (**Open/Closed**).
- If the API fails, hits a rate limit or has no key, the factory returns `NoAIProvider`, which returns a short template-based explanation. The app keeps working.
- Cache responses per plan so the same plan never calls the API twice.
- API keys go in environment variables, never in the repository.

---

## 7. Design Patterns

| Pattern | Where it is used |
|---------|------------------|
| **Strategy** (main pattern) | Interchangeable calculation formulas (Mifflin-St Jeor, Harris-Benedict) and planning algorithms (balanced, high-protein, vegetarian) |
| **Factory** | Creates the right plan generator or diet profile from the user's goal; also chooses the `LLMProvider` |
| **Repository** | `FoodRepository`, `PlanRepository`, `LogRepository` hide all queries |
| **Observer** | Alerts when a day's log goes over or under target |
| **Adapter** | Wraps each external API (USDA, Gemini) so the rest of the system sees one clean interface |
| **Builder or Decorator** (optional) | Builder assembles a meal plan; Decorator adds dietary filters |

---

## 8. SOLID and GRASP Mapping

### SOLID

| Principle | Example in this project |
|-----------|-------------------------|
| Single Responsibility | `TargetCalculator`, `PlanGenerator`, `FoodRepository` each do one job |
| Open/Closed | Add a new diet strategy or LLM provider without editing existing code |
| Liskov Substitution | Any `DietStrategy` or `LLMProvider` can replace another without breaking callers |
| Interface Segregation | Small interfaces such as `Calculable` and `Plannable` |
| Dependency Inversion | Services depend on `FoodRepository`, `DietStrategy` and `LLMProvider` abstractions |

### GRASP

| Principle | Example |
|-----------|---------|
| Information Expert | A `Meal` totals its own nutrients |
| Controller | Django views receive requests and delegate to services |
| Creator | Factories create plan generators and providers |
| Polymorphism | Diet strategies chosen by type, not by `if/else` chains |
| Low coupling, high cohesion | Each app has one focus; layers only call downward |
| Protected Variations | Strategy and provider interfaces shield services from changes in rules and vendors |

---

## 9. Non-Functional Requirements

| Attribute | Target | Tactic | How we prove it |
|-----------|--------|--------|-----------------|
| Maintainability | New diet rule added without touching old classes | Strategy, SOLID | Add one strategy in the demo with zero edits to existing files |
| Performance | Plan generated in under 3 seconds | Cached food data, indexed queries, no runtime API dependency | Timed test |
| Accuracy | Calculations match published formulas | Unit tests with known values | Test suite |
| Security | Health data visible only to its owner | Authentication, ownership checks, CSRF, hashed passwords | Tests that try to access another user's data |
| Usability | Plan generated in 3 clicks or fewer | Simple form, clear defaults | Click-count walkthrough |
| Testability | Core logic testable without the web layer | Layering, repositories, interfaces | Domain tests run with no Django request |
| Reliability | App works when the AI service is down | `NoAIProvider` fallback | Test with the provider disabled |

---

## 10. UML Diagrams (in `/docs/uml`)

| Diagram | Shows |
|---------|-------|
| Use case | User and admin goals |
| Class | Entities, services, repositories, Strategy and Factory classes |
| Sequence | Generating a meal plan, call by call across the layers |
| Layered architecture / component | The layers and their Django apps |
| Activity | The plan generation flow |
| Deployment (optional) | Browser, Django server, database |

Diagrams must match the code. Update them whenever an interface changes.

---

## 11. Project Structure

```
nutrient-meal-planner/
├── manage.py
├── config/                 # settings, urls
├── accounts/
│   ├── views.py  forms.py  templates/
│   ├── services.py
│   └── models.py           # Profile
├── nutrition/
│   ├── services.py         # TargetService
│   ├── calculators.py      # BMR strategies
│   └── tests/
├── foods/
│   ├── models.py
│   ├── repositories.py     # FoodRepository
│   └── fixtures/foods.json
├── planner/
│   ├── services.py         # PlanService
│   ├── strategies.py       # DietStrategy implementations
│   ├── factories.py
│   └── ai/                 # LLMProvider, Gemini, Ollama, NoAI
├── tracking/
│   ├── services.py
│   ├── observers.py
│   └── repositories.py
├── docs/
│   ├── uml/
│   └── report/
└── README.md
```

---

## 12. Team Responsibilities

| Part | Owner | Scope |
|------|-------|-------|
| A: Core and integration | Abdul Hadi | Project skeleton, `accounts`, `nutrition`, calculator strategies and factory, merging all code |
| B: Foods and planner | Member 1 | `foods`, `FoodRepository`, seed data, `planner`, diet strategies, plan factory, LLM provider interface |
| C: Tracking and UI | Member 2 | `tracking`, Observer alerts, all views, forms and templates |
| D: Design docs and testing | Member 3 | All UML diagrams, SOLID/GRASP write-up, NFR evidence, unit tests, report and slides |

**Ground rules**
- Agree on interfaces (`FoodRepository`, `DietStrategy`, `TargetCalculator`, `LLMProvider`) in the first two days, before anyone codes.
- One branch per person, merged through the lead. Nobody pushes directly to `main`.
- Views call services, services call repositories. Nobody skips a layer.

---

## 13. Timeline (about 14 days)

| Days | Milestone |
|------|-----------|
| 1-2 | Interfaces agreed, repo and skeleton ready, seed data format decided |
| 3-7 | Parallel build: accounts and nutrition, foods and planner, tracking and UI; diagrams drafted; test cases written |
| 8-10 | Integration, sequence and activity diagrams finalised, tests run against real code |
| 11-12 | Bug fixing, UI polish, SOLID/GRASP write-up checked against the code |
| 13-14 | Report, slides, demo rehearsal, buffer |

**If time runs short, cut in this order:** deployment diagram, Builder/Decorator, 7-day plan (keep 1-day), AI explanations. Never cut authentication, target calculation, tracking or tests.

---

## 14. Setup

```bash
git clone <repo-url>
cd nutrient-meal-planner
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env            # add SECRET_KEY, optional GEMINI_API_KEY
python manage.py migrate
python manage.py loaddata foods/fixtures/foods.json
python manage.py createsuperuser
python manage.py runserver
```

Run tests:

```bash
python manage.py test
```

---

## 15. Disclaimer

This is a university project. Targets and plans are estimates from standard formulas and are not medical or dietary advice. People with medical conditions should consult a qualified professional.