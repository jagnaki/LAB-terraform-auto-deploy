# Create README.md file with the provided content

content = """# Kalkulator API — DevOps LAB-04

Rozszerzenie projektu **LAB-03** o podejście *Infrastructure as Code* (Terraform) oraz automatyczny pipeline CI/CD wdrażający aplikację do **Azure Kubernetes Service (AKS)** po każdym commicie.

---

## 📌 Opis projektu

Aplikacja udostępnia REST API kalkulatora i demonstruje pełny cykl DevOps:

- budowanie aplikacji
- testowanie
- konteneryzacja (Docker)
- publikacja do rejestru (ACR)
- deployment do Kubernetes (AKS)
- automatyzacja (GitHub Actions)

---

## 🚀 Funkcjonalność API

| Endpoint   | Metoda | Opis |
|------------|--------|------|
| /health  | GET    | Sprawdzenie stanu aplikacji |
| /dodaj   | POST   | Dodawanie dwóch liczb |
| /odejmij | POST   | Odejmowanie dwóch liczb |
| /mnoz    | POST   | Mnożenie dwóch liczb |
| /dziel   | POST   | Dzielenie (z obsługą dzielenia przez zero) |

---

## 🏗️ Architektura

GitHub Repository
        │
        │ (CI/CD Pipeline)
        ▼
Azure Container Registry (ACR)
        │
        ▼
Azure Kubernetes Service (AKS)
        ▲
        │
Terraform (Infrastructure as Code)

---

## 📁 Struktura projektu

.
├── .github/
│   └── workflows/
│       └── ci.yml
├── infra/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── k8s/
│   └── deployment.yaml
├── app/
│   ├── main.py
│   └── test_main.py
├── Dockerfile
├── requirements.txt
└── README.md

---

## 🔧 Wymagania

- Python 3.10+
- Docker
- Terraform >= 1.0
- Azure CLI
- kubectl
- Konto Azure

---

## ☁️ Infrastruktura (Terraform)

cd infra
terraform init
terraform apply -var="acr_name=acrlab04<suffix>"

---

## ☸️ Kubernetes — pierwsze wdrożenie

az aks get-credentials --resource-group rg-lab04 --name aks-lab04
kubectl apply -f k8s/deployment.yaml

---

## 🔁 CI/CD (GitHub Actions)

Pipeline uruchamia się przy każdym push na `main`.

---

## 💻 Uruchomienie lokalne

pip install -r requirements.txt
cd app
uvicorn main:app --host 0.0.0.0 --port 8080

---

## 🧪 Testy

pip install -r requirements.txt
cd app
pytest

---

## 📡 Przykłady użycia

curl http://<EXTERNAL-IP>/health

---

## 📌 Uwagi

Projekt pokazuje kompletny pipeline DevOps.
"""

file_path = "/mnt/data/README.md"
with open(file_path, "w", encoding="utf-8") as f:
    f.write(content)

file_path