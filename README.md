# Raport z Wdrożenia Systemu MedApp w środowisku Kubernetes (Minikube)

Dokument stanowi sprawozdanie z realizacji zadania projektowego nr 2 i zawiera pełne uzasadnienie doboru obiektów, zarządzania zasobami oraz polityki bezpieczeństwa sieciowego klastra.

---

## 1. Przestrzeń nazw (Namespace)
Całość systemu została odizolowana i wdrożona w dedykowanej przestrzeni nazw o nazwie `medapp`.
*   **Uzasadnienie:** Pozwala to na pełną izolację logiczną zasobów medycznych od innych aplikacji w klastrze, łatwiejsze zarządzanie uprawnieniami (RBAC) oraz prostszą kontrolę limitów zasobów i polityk sieciowych (NetworkPolicies).

---

## 2. Dobór obiektów sterujących (Deployment vs. StatefulSet) i replikacja

### Mikrousługi Backendowe oraz Frontend (`Deployment`)
Dla serwisów: `user-service`, `appointment-service`, `booking-service`, `api-gateway` oraz `frontend` wykorzystano obiekt **Deployment**.
*   **Uzasadnienie:** Mikrousługi te są aplikacjami **bezwariowymi (stateless)**. Każda replika zachowuje się dokładnie tak samo, nie przechowuje lokalnego stanu na dysku i może zostać w dowolnym momencie zniszczona lub zastąpiona nową bez utraty spójności systemu.
*   **Ilość replik:** 
    *   Serwisy backendowe posiadają skonfigurowane **2 repliki**, co zapewnia wysoką dostępność (*High Availability*). W przypadku awarii jednego poda, ruch jest natychmiast kierowany na drugi, eliminując *Single Point of Failure*.
    *   Frontend posiada **1 replikę** z racji uruchamiania w testowym środowisku Minikube, co optymalizuje zużycie pamięci RAM środowiska lokalnego.

### Baza danych (`StatefulSet`)
Dla mikrousługi bazodanowej `mysql-db` zastosowano obiekt **StatefulSet**.
*   **Uzasadnienie:** Baza danych wymaga zachowania **stanu (stateful)**, unikalnej tożsamości sieciowej oraz stabilnego, trwałego dysku. StatefulSet gwarantuje stałą kolejność uruchamiania oraz uniemożliwia jednoczesny zapis do tego samego wolumenu przez zduplikowane procesy (uniknięcie uszkodzenia danych).
*   **Ilość replik:** Skonfigurowano **1 replikę**, ponieważ uruchamianie klastrów master-slave bazy danych wykracza poza zakres środowiska deweloperskiego Minikube i generowałoby zbędny narzut zasobów.

---

## 3. Typy usług sieciowych (Services)

*   **Serwisy Backendowe i Gateway (`ClusterIP`):** 
    Uzasadnieniem wyboru domyślnego typu `ClusterIP` jest zapewnienie bezpieczeństwa. Usługi te są dostępne wyłącznie wewnątrz klastra dla innych podów. Komunikacja nie opuszcza wewnętrznej sieci wirtualnej Kubernetes.
*   **Baza danych (`Headless Service` - `ClusterIP: None`):** 
    Zastosowany dla StatefulSet. Pozwala na bezpośrednie odpytywanie konkretnego poda bazodanowego po jego nazwie DNS (`mysql-db-0.mysql-db-service`) z pominięciem mechanizmu load balancera serwisu, co jest standardem dla baz danych.
*   **Frontend (`NodePort`):** 
    W warunkach lokalnych dystrybucji Minikube, zastosowanie `NodePort` (port `30080`) jest najbardziej niezawodnym i zalecanym sposobem wystawienia interfejsu użytkownika na maszynę hosta, zapewniając bezpośredni dostęp poprzez adres `http://<minikube-ip>:30080`. Alternatywą jest `LoadBalancer` uruchamiany przez polecenie `minikube tunnel`.

---

## 4. Przechowywanie danych (PV / PVC / StorageClass)

Trwałość danych bazy MySQL oparto o mechanizm **Dynamic Volume Provisioning**.
W manifeście zdefiniowano **PersistentVolumeClaim (PVC)** o żądanej pojemności `2Gi`. 
*   **Opis mechanizmu:** W środowisku Minikube, domyślna klasa pamięci (`standard` StorageClass oparta na sterowniku `hostPath`) automatycznie wykrywa zgłoszone zapotrzebowanie PVC i w locie tworzy fizyczny wolumen PV na dysku wirtualnym Minikube. Dzięki temu, po usunięciu poda bazy danych lub jego restartu, dane medyczne pacjentów nie ulegają skasowaniu.

---

## 5. Konfiguracja i Sekrety (ConfigMap / Secrets)

Rozdzielono jawną konfigurację od danych wrażliwych za pomocą mechanizmów Kubernetes:
*   **ConfigMap (`medapp-config`):** Przechowuje zmienne środowiskowe wspólne oraz niejawne takie jak: adresy hostów, porty usług, flagi kompatybilności Springa. Zmienne te są bezpiecznie wstrzykiwane do kontenerów za pomocą deklaracji `envFrom`.
*   **Secret (`medapp-secrets`):** Przechowuje zaszyfrowane klucze (Base64) i hasła: root pass bazy danych, klucz podpisu tokenów JWT (`JWT_SECRET`) oraz hasła pocztowe SMTP. Wstrzykiwanie odbywa się poprzez referencje `valueFrom.secretKeyRef`, dzięki czemu poświadczenia nie są widoczne w kodzie źródłowym ani w logach.

---
