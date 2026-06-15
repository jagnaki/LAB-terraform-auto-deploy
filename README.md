# LAB-04 — Infrastructure as Code (IaC) i Automatyczny Deployment

> Pełny proces CI/CD: Terraform → GitHub Actions → Azure Container Registry → Azure Kubernetes Service

---

## Spis treści

- [Cel projektu](#cel-projektu)
- [Architektura](#architektura)
- [Struktura projektu](#struktura-projektu)
- [Wymagania wstępne](#wymagania-wstępne)
- [1. Logowanie do Azure](#1-logowanie-do-azure)
- [2. Provisionowanie infrastruktury](#2-provisionowanie-infrastruktury-terraform)
- [3. Konfiguracja sekretów GitHub](#3-konfiguracja-sekretów-github)
- [4. Uruchomienie aplikacji w klastrze](#4-uruchomienie-aplikacji-lokalnie-w-klastrze)
- [5. Weryfikacja działania](#5-weryfikacja-działania)
- [6. Automatyczny deployment CI/CD](#6-automatyczny-deployment-cicd)
- [7. Zatrzymanie aplikacji](#7-zatrzymanie-aplikacji-po-testach)
- [8. Ponowne uruchomienie](#8-ponowne-uruchomienie-aplikacji)
- [9. Czego NIE robić](#9-ważne--czego-nie-robić)
- [10. Zarządzanie sekretami](#10-uwaga-dotycząca-sekretów)
- [Podsumowanie](#podsumowanie)

---

## Cel projektu

Wdrożenie pełnego procesu CI/CD z użyciem:

| Narzędzie | Rola |
|---|---|
| **Terraform** | Tworzenie infrastruktury (ACR + AKS) |
| **GitHub Actions** | Automatyczny pipeline (Build → Test → Push → Deploy) |
| **Azure Kubernetes Service (AKS)** | Uruchamianie aplikacji kontenerowej |
| **Azure Container Registry (ACR)** | Przechowywanie obrazów Docker |

Każdy push do `main` automatycznie:
1. Buduje aplikację
2. Uruchamia testy
3. Buduje obraz Docker
4. Wysyła obraz do ACR
5. Aktualizuje deployment w AKS

---

## Architektura

```
┌─────────────────────┐
│    GitHub Actions   │  ← push do main triggeruje pipeline
└──────────┬──────────┘
           │  build & test
┌──────────▼──────────┐
│    Docker Build     │
└──────────┬──────────┘
           │  docker push
┌──────────▼──────────┐
│  Azure Container    │
│  Registry  (ACR)    │
└──────────┬──────────┘
           │  kubectl rollout
┌──────────▼──────────┐
│  Azure Kubernetes   │
│  Service   (AKS)    │
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│  FastAPI Application│
└─────────────────────┘
```

---

## Struktura projektu

```
.
├── .github/
│   └── workflows/
│       └── ci.yml          # pipeline CI/CD
├── infra/                  # Terraform
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── k8s/                    # Kubernetes manifests
│   └── deployment.yaml
├── app/                    # aplikacja FastAPI
├── Dockerfile
└── README.md
```

---

## Wymagania wstępne

Przed rozpoczęciem upewnij się, że masz zainstalowane:

- [Terraform](https://developer.hashicorp.com/terraform/install) `>= 1.3`
- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Docker](https://docs.docker.com/get-docker/)

---

## 1. Logowanie do Azure

Przed jakimikolwiek operacjami na infrastrukturze musisz zalogować się do Azure CLI.

### Logowanie interaktywne (zalecane, działa też przez SSH)

```bash
az login --use-device-code
```

Komenda wyświetli kod i link, np.:

```
To sign in, use a web browser to open the page https://microsoft.com/devicelogin
and enter the code XXXXXXXXX to authenticate.
```

Otwórz link w przeglądarce, wpisz kod i zaloguj się kontem Azure. Po chwili terminal potwierdzi logowanie.

> **Kiedy używać `--use-device-code`?**
> Gdy pracujesz przez SSH, WSL, terminal bez GUI lub gdy standardowy `az login` (który otwiera przeglądarkę automatycznie) nie działa w Twoim środowisku.

### Weryfikacja logowania

```bash
az account show
```

Upewnij się, że aktywna subskrypcja to właściwa (zwróć uwagę na pole `name` i `id`).

### Przełączenie subskrypcji (jeśli masz kilka)

```bash
az account list --output table
az account set --subscription "<SUBSCRIPTION_ID>"
```

---

## 2. Provisionowanie infrastruktury (Terraform)

### Inicjalizacja

```bash
cd infra
terraform init
```

### Generowanie unikalnej nazwy ACR

Nazwa ACR musi być **globalnie unikalna**, zawierać tylko litery i cyfry, od 5 do 50 znaków:

```bash
openssl rand -hex 4
# Przykładowy wynik: a3f9c12b
# Użyj jako nazwy: acra3f9c12b
```

> **Zapisz tę nazwę** — będzie potrzebna w GitHub Secrets (`ACR_LOGIN_SERVER`).

### Podgląd planowanych zmian (zalecane)

```bash
terraform plan -var="acr_name=acr<wynik_openssl>"
```

Przykład z prawdziwą wartością:

```bash
terraform plan -var="acr_name=acra3f9c12b"
```

### Uruchomienie provisioningu

```bash
terraform apply -var="acr_name=acra3f9c12b"
```

Po zakończeniu zostaną utworzone:
- **Resource Group** `rg-lab04`
- **Azure Container Registry** (ACR)
- **Azure Kubernetes Service** (AKS)

---

## 3. Konfiguracja sekretów GitHub

Przejdź do: **GitHub → Settings → Secrets and variables → Actions → New repository secret**

> ⚠️ **Uwaga:** To podejście używa tokenu ACR zamiast Service Principal. Jest to rozwiązanie zalecane dla subskrypcji studenckich, które nie posiadają uprawnień do tworzenia aplikacji w Azure AD (`az ad sp create-for-rbac`).

---

### Wymagane sekrety

| Secret | Opis | Skąd pobrać | Komenda |
|---|---|---|---|
| `ACR_LOGIN_SERVER` | np. `acra3f9c12b.azurecr.io` | Wynik `terraform apply` lub portal Azure | `az acr show --name <acr_name> --query loginServer -o tsv` |
| `ACR_USERNAME` | Nazwa tokenu ACR (np. `lab04token`) | Pole `username` z wyniku `az acr token create` | *(patrz sekcja "Tworzenie tokenu ACR" poniżej)* |
| `ACR_PASSWORD` | Hasło tokenu ACR | Pole `passwords[0].value` z wyniku `az acr token create` | *(patrz sekcja "Tworzenie tokenu ACR" poniżej)* |
| `KUBE_CONFIG` | Plik kubeconfig zakodowany w base64 | Klaster AKS | *(patrz sekcja "Generowanie KUBE_CONFIG" poniżej)* |

---

### Tworzenie tokenu ACR

Token ACR daje dostęp tylko do rejestru — nie wymaga uprawnień Azure AD.

```bash
az acr token create \
  --name lab04token \
  --registry <acr_name> \
  --scope-map _repositories_push \
  --output json
```

Wynik JSON zawiera potrzebne wartości:

```json
{
  "credentials": {
    "passwords": [
      { "name": "password1", "value": "..." }  // → ACR_PASSWORD
    ],
    "username": "lab04token"                    // → ACR_USERNAME
  }
}
```

> **Uwaga:** `ACR_LOGIN_SERVER` to `<acr_name>.azurecr.io` — tę samą nazwę podałeś podczas `terraform apply`.

---

### Generowanie KUBE_CONFIG (base64)

**Linux:**

```bash
az aks get-credentials \
  --resource-group rg-lab04 \
  --name aks-lab04 \
  --file - | base64 -w 0
```

**macOS** (brak flagi `-w 0`):

```bash
az aks get-credentials \
  --resource-group rg-lab04 \
  --name aks-lab04 \
  --file - | base64
```

Skopiuj cały wynik (jedna linia) jako wartość sekretu `KUBE_CONFIG`.

---

### Konfiguracja logowania do ACR w pipeline

W pliku `.github/workflows/ci.yml` zastąp krok logowania do ACR następującym:

```yaml
- name: Log in to ACR
  uses: docker/login-action@v3
  with:
    registry: ${{ secrets.ACR_LOGIN_SERVER }}
    username: ${{ secrets.ACR_USERNAME }}
    password: ${{ secrets.ACR_PASSWORD }}
```

---

## 4. Uruchomienie aplikacji lokalnie w klastrze

> ⚠️ **Uwaga:** `kubectl apply` zadziała tylko wtedy, gdy obraz Docker **już istnieje w ACR**. Upewnij się, że pipeline CI/CD wykonał się przynajmniej raz (lub wypchnij obraz ręcznie), zanim uruchomisz poniższe komendy — inaczej pody będą w stanie `ImagePullBackOff`.

### Sprawdzenie czy obraz istnieje w ACR

```bash
az acr repository show-tags \
  --name <nazwa_acr> \
  --repository app \
  --output table
```

Jeśli lista tagów jest pusta lub komenda zwraca błąd `repository does not exist` — obraz nie
istnieje i należy go wypchnąć ręcznie przed pierwszym deploymentem.

### Ręczne wypchnięcie obrazu (jeśli pipeline jeszcze nie działał)
Pamiętaj żeby wykonać te komendy z katalogu głównego repozytorium (tam gdzie jest Dockerfile), nie z infra/.

```bash
# Zaloguj się do ACR przez token
docker login <nazwa_acr>.azurecr.io \
  --username lab04token \
  --password "<haslo_do_tokena>"

# Zbuduj i wypchnij z tymczasowym tagiem
docker build -t <nazwa_acr>.azurecr.io/app:manual .
docker push <nazwa_acr>.azurecr.io/app:manual
```

Następnie tymczasowo podmień `image: placeholder` w `k8s/deployment.yaml`:

```yaml
containers:
  - name: app
    image: <nazwa_acr>.azurecr.io/app:manual   # ← zmień z placeholder
```

### Pobierz credentials klastra

```bash
az aks get-credentials \
  --resource-group rg-lab04 \
  --name aks-lab04
```

### Zastosuj manifest Kubernetes

```bash
kubectl apply -f k8s/deployment.yaml
```

> **Uwaga:** Przed commitem przywróć `image: placeholder` w `k8s/deployment.yaml`.
> Pipeline CI/CD nadpisuje tag obrazu przez `kubectl set image` przy każdym deployu,
> więc wartość w pliku ma znaczenie tylko przy pierwszym `kubectl apply`.

Po pierwszym udanym przebiegu pipeline CI/CD przejmie aktualizację obrazu przez
`kubectl set image` — tag `manual` nie będzie już potrzebny.

---

## 5. Weryfikacja działania

### Stan podów

```bash
kubectl get pods
```

Pody powinny mieć status `Running`.

### Adres External IP serwisu

```bash
kubectl get svc app-svc
```

> **Uwaga:** Przyznanie External IP może potrwać kilka minut. Jeśli widzisz `<pending>`, poczekaj i ponów komendę.

### Test endpointu health

```bash
curl http://<EXTERNAL-IP>/health
```

---

## 6. Automatyczny deployment (CI/CD)

Po każdej zmianie wypchnij kod do `main`:

```bash
git add .
git commit -m "update app"
git push origin main
```

Pipeline GitHub Actions uruchomi kolejno:

| Krok | Akcja | Opis |
|---|---|---|
| 1 | **Build** | Kompilacja i budowanie aplikacji |
| 2 | **Test** | Uruchomienie testów jednostkowych |
| 3 | **Docker Push** | Budowanie i wysyłanie obrazu do ACR |
| 4 | **Deploy** | Aktualizacja deployment w AKS (`kubectl rollout`) |

---

## 7. Zatrzymanie aplikacji po testach

### Zalecane — skalowanie do zera replik

```bash
kubectl scale deployment app --replicas=0
```

Efekt:
- aplikacja przestaje działać
- AKS i ACR nadal istnieją
- sekrety GitHub pozostają ważne — **nie trzeba ich zmieniać**

### Alternatywa — usunięcie deploymentu

```bash
# Usunięcie
kubectl delete -f k8s/deployment.yaml
```

---
## 8. Ponowne uruchomienie aplikacji

Jeśli infrastruktura nadal istnieje (nie wykonywałeś `terraform destroy`):

```bash
# 1. Upewnij się że masz kubeconfig
az aks get-credentials --resource-group rg-lab04 --name aks-lab04

# 2. Zastosuj manifest (deployment + serwis)
kubectl apply -f k8s/deployment.yaml

# 3. Podmień obraz (plik YAML zawiera placeholder)
kubectl set image deployment/app app=<nazwa_acr>.azurecr.io/app:manual

# 4. Sprawdź czy pody działają
kubectl get pods -w

# 5. Sprawdź publiczny IP
kubectl get services
```

Poczekaj aż pody mają status `Running`, a `EXTERNAL-IP` przy `app-svc` przestanie być
`<pending>` — wtedy aplikacja jest dostępna pod tym adresem.

Nie trzeba zmieniać żadnych sekretów.

## 9. WAŻNE - czego NIE robić

> **Nie uruchamiaj `terraform destroy`**, jeśli chcesz zachować:
> - GitHub Secrets
> - Azure Container Registry (ACR)
> - Azure Kubernetes Service (AKS)
> - konfigurację CI/CD

Jeśli `terraform destroy` zostanie wykonany:
- ACR i AKS są usuwane i tworzone od nowa
- generowane są nowe dane dostępowe
- **konieczna jest aktualizacja wszystkich sekretów w GitHub**
---

## Podsumowanie

| Narzędzie | Rola |
|---|---|
| `terraform` | Provisionuje infrastrukturę (ACR + AKS + Resource Group) |
| `github actions` | Automatyzuje pipeline: build → test → push → deploy |
| `azure acr` | Przechowuje wersjonowane obrazy Docker |
| `azure aks` | Uruchamia kontenery i udostępnia aplikację |
| `kubectl scale` | Zatrzymuje aplikację bez niszczenia infrastruktury |
