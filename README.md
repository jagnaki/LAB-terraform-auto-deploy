# LAB-04 — Infrastruktura jako kod (IaC) i Automatyczny Deployment

## Cel zadania
Celem projektu jest wdrożenie automatycznego procesu CI/CD przy użyciu **Terraform** oraz **GitHub Actions**. Infrastruktura (ACR oraz AKS) jest zarządzana za pomocą kodu, a każdy `git push` na gałąź `main` automatycznie buduje aplikację, testuje ją, wypycha obraz do rejestru i aktualizuje serwis w klastrze Kubernetes.

## Architektura
1. **Terraform**: Automatyczne provisionowanie grupy zasobów, ACR i AKS.
2. **GitHub Actions**: Pipeline realizujący pełny cykl życia aplikacji (Build → Test → Push → Rollout).
3. **AKS**: Klaster Kubernetes przyjmujący zaktualizowany obraz kontenera.

## Struktura projektu
```text
.
├── .github/workflows/ci.yml  # Pipeline CI/CD
├── infra/                    # Kod Terraform
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── k8s/                      # Manifesty Kubernetes
│   └── deployment.yaml
├── app/                      # Kod aplikacji (FastAPI)
└── Dockerfile
```

## Instrukcja uruchomienia

### 1. Provisionowanie infrastruktury (Terraform)
Wejdź do folderu `infra` i uruchom:

```bash
terraform init
openssl rand -hex 4 #losowy ciąg znaków
terraform apply -var="acr_name=acr<losowy ciąg znaków>"
```

### 2. Przygotowanie sekretów GitHub
Aby pipeline miał dostęp do zasobów, dodaj poniższe sekrety w:
**GitHub Settings → Secrets and variables → Actions**

| Sekret | Wartość |
|--------|----------|
| AZURE_CLIENT_ID | `terraform output acr_admin_username` |
| AZURE_CLIENT_SECRET | `terraform output -raw acr_admin_password` |
| AZURE_TENANT_ID | `az account show --query tenantId -o tsv` |
| AZURE_SUBSCRIPTION_ID | `az account show --query id -o tsv` |
| ACR_LOGIN_SERVER | `<twoja_nazwa_acr>.azurecr.io` |
| KUBE_CONFIG | `az aks get-credentials --resource-group rg-lab04 --name aks-lab04 --file - \| base64 -w 0` |

### 3. Pierwsze wdrożenie aplikacji

```bash
# Pobranie poświadczeń lokalnie
az aks get-credentials --resource-group rg-lab04 --name aks-lab04

# Zastosowanie manifestów
kubectl apply -f k8s/deployment.yaml
```

## Weryfikacja działania

Sprawdź pody:
```bash
kubectl get pods
```

Pobierz adres serwisu:
```bash
kubectl get svc app-svc
```

Test API:
```bash
curl http://<EXTERNAL-IP>/health
```

## Test automatycznego deployu
Wprowadź zmianę w kodzie aplikacji i wykonaj:

```bash
git add .
git commit -m "Test auto-deploy"
git push origin main
```

Pipeline automatycznie zaktualizuje deployment w AKS.

## Uwaga
Projekt korzysta z lokalnego pliku `terraform.tfstate`, który powinien być dodany do `.gitignore`.
