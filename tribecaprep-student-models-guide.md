# Tribeca Prep — Student Profile System

## What This Is

A **production-ready Student Profile System** built on [Boxel](https://app.boxel.ai) using Glimmer TypeScript (`.gts`). It provides the foundational data layer that all other LMS modules (Classroom Dashboard, Daily Reports, Learning Goals, etc.) depend on.

The system has two tiers:

| Card | Realm | Who sees it | Purpose |
|------|-------|-------------|---------|
| **Student** (stub) | LMS (operational) | All teaching staff | Day-to-day student info: name, grade, flags, tags |
| **StudentFullProfile** | Sensitive | HoS + Front Desk only | Medical, family, custody, financial, IEP/504 |

---

## File Structure

```
tribecaprep-student-models/
├── enums.gts                  # Shared enums: GradeLevel, MasteryLevel, Subject, ObservationCategory, Priority
├── shared-fields.gts          # Shared FieldDefs: StudentTag, LocationStatus, TribecaPrepCardInfo
├── lms-fields.gts             # LMS FieldDefs: ParentInfo, Address, RelationshipType
├── profile-sections.gts       # Sensitive sections: Identity, Medical, Family, Custody, Financial, IEP
├── student.gts                # Student CardDef (operational stub)
├── student-full-profile.gts   # StudentFullProfile CardDef (sensitive)
├── student-directory.gts      # StudentDirectory AppCard (search & browse)
├── Student/                   # Student instance JSON data
│   ├── emma-chen.json
│   ├── marcus-johnson.json
│   └── sofia-martinez.json
├── StudentFullProfile/        # Full profile instance JSON data
│   ├── emma-chen-001.json
│   ├── marcus-johnson-002.json
│   └── sofia-martinez-003.json
└── StudentDirectory/
    └── main.json
```

---

## How to Deploy to Your Boxel Workspace

### Prerequisites

- Node.js 18+
- [Boxel CLI](https://github.com/cardstack/boxel-cli) installed (`npm install -g @anthropic/boxel-cli` or use local build)
- A Boxel account with profile configured (`boxel profile add`)

### Steps

```bash
# 1. Create a new workspace on Boxel
boxel create <your-endpoint> "My LMS Workspace"

# 2. Push all files to the workspace
cd tribecaprep-student-models
boxel push . https://app.boxel.ai/<your-username>/<workspace-name>/ --force

# 3. Force reindex to compile all types
REALM_SERVER_URL=https://app.boxel.ai boxel touch . --all

# 4. Open your workspace in the browser to verify
# Go to: https://app.boxel.ai/<your-username>/<workspace-name>/
```

---

## How to Use Student in Your Module

### 1. Link TO a Student (most common)

When your module needs to reference a student (e.g., a LearningGoal belongs to a student), use `linksTo`:

```ts
// my-module.gts
import { CardDef, field, linksTo } from 'https://cardstack.com/base/card-api';
import { Student } from './student';

export class LearningGoal extends CardDef {
  static displayName = 'Learning Goal';

  // One-way link: this goal belongs to ONE student
  @field student = linksTo(() => Student);

  // ... your other fields
}
```

**In JSON data**, the link looks like:
```json
{
  "relationships": {
    "student": {
      "links": {
        "self": "../Student/emma-chen"
      }
    }
  }
}
```

> **Important**: Links are always **one-way** (downward). A LearningGoal links TO a Student, but Student does NOT link back. To find all goals for a student, use a query filter.

### 2. Link to MANY Students

For cards that reference multiple students (e.g., a Classroom roster):

```ts
import { CardDef, field, linksToMany } from 'https://cardstack.com/base/card-api';
import { Student } from './student';

export class Classroom extends CardDef {
  @field students = linksToMany(() => Student);
}
```

### 3. Display a Student in Your Template

The Student card has **4 display formats** you can use:

```hbs
{{! Full view — shows all details }}
<@fields.student @format='isolated' />

{{! Compact card — avatar + name + grade + badge pills }}
<@fields.student @format='embedded' />

{{! Minimal — avatar + displayName + grade tag, for grids/rosters }}
<@fields.student @format='fitted' />

{{! Inline mention — tiny purple pill with avatar + name, for rich text }}
<@fields.student @format='atom' />
```

### 4. Access Student Fields in Your Code

When you have a `linksTo` Student, you can access all its fields:

```ts
// In a computed field:
@field summary = contains(StringField, {
  computeVia: function (this: LearningGoal) {
    const s = this.student;
    if (!s) return '';
    return `${s.fullName} (${s.gradeLevel?.value || 'N/A'})`;
  },
});
```

**Available Student fields:**

| Field | Type | Example Value |
|-------|------|---------------|
| `studentId` | string | `"TP-2022-0156"` |
| `firstName` | string | `"Emma"` |
| `lastName` | string | `"Chen"` |
| `preferredName` | string | `"Emmy"` |
| `dateOfBirth` | string | `"2016-03-15"` |
| `gradeLevel.value` | string | `"3rd"` |
| `photoUrl` | string | URL or null |
| `active` | boolean | `true` |
| `hasIEP` | boolean | `true` |
| `has504` | boolean | `false` |
| `hasAllergy` | boolean | `true` |
| `hasMedication` | boolean | `true` |
| `hasCustodyAlert` | boolean | `false` |
| `displayName` | computed | `"Emmy C."` |
| `fullName` | computed | `"Emma Chen"` |
| `initials` | computed | `"EC"` |
| `age` | computed | `10` |
| `tagSummary` | computed | `"📋 IEP · ⚠️ Allergy · 💊 Meds"` |
| `sortName` | computed | `"Chen, Emma"` |

### 5. Use Shared Enums

Import shared enums from `./enums` for consistent data across all modules:

```ts
import {
  GradeLevel,       // K, 1st, 2nd, ... 12th
  MasteryLevel,     // emerging, developing, approaching, meeting, mastered
  Subject,          // mathematics, ela, science, social-studies, art, music, pe
  ObservationCategory, // academic, social-emotional, physical, communication
  Priority,         // low, medium, high, critical
} from './enums';

export class LearningGoal extends CardDef {
  @field domain = contains(Subject);
  @field currentLevel = contains(MasteryLevel);
  @field priority = contains(Priority);
}
```

**In JSON**, enums are stored as `{ "value": "string" }`:
```json
{
  "domain": { "value": "mathematics" },
  "currentLevel": { "value": "approaching" },
  "priority": { "value": "high" }
}
```

Each enum has a **color-coded embedded view** using Tribeca Prep's brand palette:
- MasteryLevel: red → yellow → purple → teal → dark teal (5 levels)
- Subject: coral (math), purple (ELA), teal (science), amber (social studies)
- Priority: gray (low) → purple (medium) → amber (high) → red (critical)

---

## How to Create New Student Data

### Student Stub (operational)

Create a new JSON file in `Student/` directory:

```json
{
  "data": {
    "type": "card",
    "attributes": {
      "studentId": "TP-2025-0301",
      "firstName": "Jamie",
      "lastName": "Chen",
      "preferredName": null,
      "gradeLevel": { "value": "2nd" },
      "dateOfBirth": "2018-05-20",
      "photoUrl": null,
      "active": true,
      "currentLocation": {
        "location": "Classroom 2A",
        "status": { "value": "in-class" },
        "since": "2026-02-27T08:15:00Z"
      },
      "hasIEP": true,
      "has504": false,
      "hasAllergy": true,
      "hasMedication": false,
      "hasCustodyAlert": false,
      "tags": [
        {
          "label": "Math Intervention",
          "color": "#e05d50",
          "tagType": { "value": "program" }
        }
      ],
      "cardInfo": {
        "title": "Jamie Chen",
        "description": "2nd grade student with IEP",
        "thumbnailURL": null,
        "notes": null
      }
    },
    "meta": {
      "adoptsFrom": {
        "module": "../student",
        "name": "Student"
      }
    }
  }
}
```

### StudentFullProfile (sensitive)

Create a new JSON file in `StudentFullProfile/` directory. See existing files for the full structure — it includes `identity`, `medical`, `family`, `custody`, `financial`, and `iep` sections.

The `studentStub` relationship links back to the operational stub:
```json
{
  "relationships": {
    "studentStub": {
      "links": {
        "self": "../Student/jamie-chen"
      }
    }
  }
}
```

---

## Architecture Notes

### Why Two Cards?

The **Student** stub contains only what teaching staff need daily (name, grade, flags). The **StudentFullProfile** contains sensitive data (medical details, family finances, custody arrangements) restricted to authorized personnel.

Other modules should **always link to Student** (the stub), never to StudentFullProfile.

### Import Map

```
enums.gts ◄── student.gts ◄── student-directory.gts
                    ▲
shared-fields.gts ──┘
                    ▲
lms-fields.gts ────┘

enums.gts ◄── profile-sections.gts ◄── student-full-profile.gts
                                              │
                                              └─► linksTo Student (from student.gts)
```

### Query Pattern (Finding Cards by Student)

To find all LearningGoals for a specific student, use a query-backed `linksToMany`:

```ts
@field goals = linksToMany(() => LearningGoal, {
  query: {
    filter: {
      type: { module: './learning-goal', name: 'LearningGoal' },
      eq: { 'student.id': studentId }
    }
  }
});
```

### Boxel Card Formats Quick Reference

| Format | Size | Use Case |
|--------|------|----------|
| `isolated` | Full page | Viewing a single card in detail |
| `embedded` | Medium card | Inside another card's view, lists |
| `fitted` | Small tile | Grids, rosters, dashboards |
| `atom` | Inline pill | Rich text mentions, compact references |

---

## Existing Sample Data

| Student | Grade | Flags | Notes |
|---------|-------|-------|-------|
| Emma Chen ("Emmy") | 3rd | IEP, Allergy, Medication | Peanut allergy (severe), reading IEP |
| Marcus Johnson | 5th | 504, Medication, Custody Alert | Type 1 diabetes, custody restrictions |
| Sofia Martinez | K | Allergy | Lactose intolerant (mild) |

---

## Troubleshooting

**Cards not rendering?**
```bash
boxel doctor repair-realm https://app.boxel.ai/<user>/<workspace>/
```

**Types not updating after code change?**
```bash
REALM_SERVER_URL=https://app.boxel.ai boxel touch . --all
```

**Index missing?**
```bash
boxel doctor force-reindex .
```
