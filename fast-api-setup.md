Run the following commands in the project terminal one by one

python3 -m venv .venv
source .venv/bin/activate
pip install 'fastapi[all]' 'uvicorn[standard]'

---

Next, use this prompt & execute the agent:

Setup this folder structure for me and setup a root level /health endpoint to let me know the API is operational. Also please make sure that "/" returns "API is running"


├── app/
│   ├── api/
│   │   ├── v1/
│   │   │   └── routes.py
│   │
│   ├── services/
│   │   └── example_service.py
│   │
│   ├── core/
│   │   └── config.py
│   │
│   ├── models/
│   ├── schemas/
│   │
│   └── main.py
│
├── .venv/
├── requirements.txt
└── README.md

