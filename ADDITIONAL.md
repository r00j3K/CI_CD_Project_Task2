## 1. Zaawansowane mechanizmy k8s (Punkt 2 - Wyższa Ocena)

### a) Ograniczenie zasobów (Resource Requests / Limits)
Dla każdego kontenera precyzyjnie określono zapotrzebowanie na zasoby:
*   Aplikacje Spring Boot (Java 21) otrzymały limit `512Mi` pamięci RAM oraz żądanie startowe `384Mi`. Zapewnia to stabilny start maszyny JVM (która pobiera do 75% alokacji dzięki flagom `MaxRAMPercentage`).
*   Dzięki limitom, żaden pod nie doprowadzi do awarii całego węzła z powodu wycieku pamięci (ochrona przed błędem OOMKilled).

### b) Polityka sieciowa (NetworkPolicy)
Wdrożono izolację sieciową warstwy danych za pomocą obiektu `NetworkPolicy` o nazwie `database-policy`.
*   **Działanie:** Baza danych `mysql-db` przyjmuje ruch sieciowy **wyłącznie** z podów, które posiadają etykiety (labels) należące do backendu: `user-service`, `appointment-service`, `booking-service`.
*   **Bezpieczeństwo:** Nawet w przypadku potencjalnego przejęcia kontenera frontowego przez intruza, napastnik nie ma możliwości nawiązania bezpośredniego połączenia z bazą MySQL z pominięciem warstwy API.

### c) Sterowanie rozmieszczeniem podów (Pod Anti-Affinity)
Dla krytycznych usług (np. `user-service`) zastosowano regułę **PodAntiAffinity** (`preferredDuringSchedulingIgnoredDuringExecution`).
*   **Działanie:** Mechanizm ten instruuje scheduler Kubernetesa, aby w miarę możliwości unikał umieszczania replik tej samej usługi na tym samym węźle fizycznym (`topologyKey: kubernetes.io/hostname`). W środowisku produkcyjnym wielowęzłowym gwarantuje to, że awaria jednego serwera fizycznego nie wyłączy obu replik danej mikrousługi naraz.

## 2. Dostęp do systemu z zewnątrz klastra (Ingress / API Gateway)

Do koordynacji i kontroli ruchu przychodzącego z zewnątrz klastra do systemu MedApp wykorzystano duet: **Kubernetes Ingress (Nginx Ingress Controller)** oraz wewnętrzny **Spring Cloud API Gateway**.

### Uzasadnienie wyboru Ingress:
1. **Pojedynczy punkt wejścia (Single Public IP):** Zamiast wystawiać każdą usługę (Frontend, API Gateway) na osobnych portach (co wymuszałoby stosowanie uciążliwych i niebezpiecznych usług typu NodePort lub generowanie kosztów za wiele publicznych LoadBalancerów), Ingress udostępnia całą aplikację pod jednym adresem IP.
2. **Routing oparty na ścieżkach (Path-based routing):** Ingress analizuje URI zapytania sieciowego:
   * Reguła `/` kieruje użytkownika do statycznych zasobów aplikacji webowej (`frontend-service`).
   * Reguła `/api` przechwytuje żądania asynchroniczne (Axios z Reacta) i przesyła je bezpiecznie do warstwy backendowej (`api-gateway-service`).
3. **Izolacja topologii sieci:** Użytkownik końcowy w przeglądarce widzi spójną aplikację pod jednym adresem. Nie wie i nie musi wiedzieć, że wewnątrz klastra działa podział na frontend i 4 niezależne mikrousługi.

### Rola Spring Cloud API Gateway w klastrze:
Mimo istnienia Ingressa, zachowano w architekturze mikrousługę `api-gateway`. Spełnia ona zadania specyficzne dla logiki biznesowej aplikacji, których Ingress nie potrafi zrealizować:
* **Przetwarzanie i walidacja tokenów JWT:** Odciąża mikrousługi z konieczności ponownej autoryzacji użytkownika przy każdym zapytaniu.
* **Agregacja i zaawansowany Load Balancing:** Dynamicznie rozkłada ruch pomiędzy dwoma replikami `user-service`, `booking-service` oraz `appointment-service` wewnątrz dedykowanej sieci klastra.
