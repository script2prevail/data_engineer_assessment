# Data Engineering Assessment

Welcome!  
This exercise evaluates your core **data-engineering** skills:

| Competency | Focus                                                         |
| ---------- | ------------------------------------------------------------- |
| SQL        | relational modelling, normalisation, DDL/DML scripting        |
| Python ETL | data ingestion, cleaning, transformation, & loading (ELT/ETL) |

---

## 0 Prerequisites & Setup

> **Allowed technologies**

- **Python ≥ 3.8** – all ETL / data-processing code
- **MySQL 8** – the target relational database
- **Pydantic** – For data validation
- List every dependency in **`requirements.txt`** and justify selection of libraries in the submission notes.

---

## 1 Clone the skeleton repo

```
git clone https://github.com/100x-Home-LLC/data_engineer_assessment.git
```

✏️ Note: Rename the repo after cloning and add your full name.

**Start the MySQL database in Docker:**

```
docker-compose -f docker-compose.initial.yml up --build -d
```

- Database is available on `localhost:3306`
- Credentials/configuration are in the Docker Compose file
- **Do not change** database name or credentials

For MySQL Docker image reference:
[MySQL Docker Hub](https://hub.docker.com/_/mysql)

---

### Problem

- You are provided with a raw JSON file containing property records is located in data/
- Each row relates to a property. Each row mixes many unrelated attributes (property details, HOA data, rehab estimates, valuations, etc.).
- There are multiple Columns related to this property.
- The database is not normalized and lacks relational structure.
- Use the supplied Field Config.xlsx (in data/) to understand business semantics.

### Task

- **Normalize the data:**

  - Develop a Python ETL script to read, clean, transform, and load data into your normalized MySQL tables.
  - Refer the field config document for the relation of business logic
  - Use primary keys and foreign keys to properly capture relationships

- **Deliverable:**
  - Write necessary python and sql scripts
  - Place your scripts in `src/`
  - The scripts should take the initial json to your final, normalized schema when executed
  - Clearly document how to run your script, dependencies, and how it integrates with your database.

---

## Submission Guidelines

- Edit the section to the bottom of this README with your solutions and instructions for each section at the bottom.
- Ensure all steps are fully **reproducible** using your documentation
- DO NOT MAKE THE REPOSITORY PUBLIC. ANY CANDIDATE WHO DOES IT WILL BE AUTO REJECTED.
- Create a new private repo and invite the reviewer https://github.com/mantreshjain and https://github.com/siddhuorama

---

**Good luck! We look forward to your submission.**

## Solutions and Instructions (Filed by Candidate)

**Document your solution here:**

**SQL Script**
CREATE TABLE property (
    property_id VARCHAR(50) PRIMARY KEY,
    MLSNumber VARCHAR(50),
    YearBuilt INT,
    LotSize FLOAT,
    PropertyStatus VARCHAR(50)
);

CREATE TABLE valuation (
    id INT AUTO_INCREMENT PRIMARY KEY,
    property_id VARCHAR(50),
    valuation_amount DECIMAL(15,2),
    valuation_date DATE,
    FOREIGN KEY (property_id) REFERENCES property(property_id)
);

CREATE TABLE hoa (
    id INT AUTO_INCREMENT PRIMARY KEY,
    property_id VARCHAR(50),
    hoa_fee DECIMAL(10,2),
    hoa_frequency VARCHAR(20),
    FOREIGN KEY (property_id) REFERENCES property(property_id)
);




**Python Script for Extracting Data**
import json
import mysql.connector
from pydantic import BaseModel
from typing import Optional
from datetime import datetime

# ----------------------------
# Pydantic Models
# ----------------------------

class PropertyModel(BaseModel):
    property_id: str
    MLSNumber: Optional[str]
    YearBuilt: Optional[int]
    LotSize: Optional[float]
    PropertyStatus: Optional[str]
    valuation_amount: Optional[float]
    valuation_date: Optional[str]
    hoa_fee: Optional[float]
    hoa_frequency: Optional[str]


# ----------------------------
# DB Connection
# ----------------------------

conn = mysql.connector.connect(
    host="localhost",
    user="db_user",
    password="6equj5_db_user",
    database="assessment"
)
cursor = conn.cursor()


# ----------------------------
# INSERT FUNCTIONS
# ----------------------------

def insert_property(p):
    cursor.execute("""
        INSERT INTO property (property_id, MLSNumber, YearBuilt, LotSize, PropertyStatus)
        VALUES (%s, %s, %s, %s, %s)
    """, (p.property_id, p.MLSNumber, p.YearBuilt, p.LotSize, p.PropertyStatus))


def insert_valuation(p):
    if not p.valuation_amount:
        return
    date_val = None
    if p.valuation_date:
        date_val = datetime.strptime(p.valuation_date, "%Y-%m-%d").date()

    cursor.execute("""
        INSERT INTO valuation (property_id, valuation_amount, valuation_date)
        VALUES (%s, %s, %s)
    """, (p.property_id, p.valuation_amount, date_val))


def insert_hoa(p):
    if not p.hoa_fee:
        return

    cursor.execute("""
        INSERT INTO hoa (property_id, hoa_fee, hoa_frequency)
        VALUES (%s, %s, %s)
    """, (p.property_id, p.hoa_fee, p.hoa_frequency))


# ----------------------------
# MAIN ETL LOGIC
# ----------------------------

def run_etl():

    with open("data/raw.json") as f:
        rows = json.load(f)

    for raw in rows:

        # Validate using Pydantic
        cleaned = PropertyModel(**raw)

        # Load into normalized tables
        insert_property(cleaned)
        insert_valuation(cleaned)
        insert_hoa(cleaned)

    conn.commit()
    print("ETL Completed Successfully!")


if __name__ == "__main__":
    run_etl()
