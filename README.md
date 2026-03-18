# 💰 PiggyMetrics — Microservices Financiers Spring Cloud

Application de gestion financière personnelle décomposée en microservices
indépendants avec Spring Cloud. Chaque service a sa propre base de données,
ses propres responsabilités et peut être déployé, scalé et mis à jour
indépendamment des autres — c'est la promesse des microservices en pratique.

---

## Décomposition en services

```
  ┌──────────────────────────────────────────────────────────────┐
  │                        Client (Browser)                      │
  └───────────────────────────┬──────────────────────────────────┘
                              │
                              ▼
  ┌──────────────────────────────────────────────────────────────┐
  │                    API Gateway (Zuul)                        │
  │   /account/** → Account Service                             │
  │   /statistics/** → Statistics Service                       │
  │   /notification/** → Notification Service                   │
  └──────┬───────────────────┬─────────────────────┬────────────┘
         │                   │                     │
         ▼                   ▼                     ▼
  ┌─────────────┐   ┌──────────────────┐   ┌──────────────────┐
  │   Account   │   │   Statistics     │   │  Notification    │
  │   Service   │   │   Service        │   │  Service         │
  │             │   │                  │   │                  │
  │ MongoDB     │   │ MongoDB          │   │ MongoDB          │
  │ (accounts)  │   │ (time series)    │   │ (settings)       │
  └─────────────┘   └──────────────────┘   └──────────────────┘
         │                   ▲
         │   publishes       │  consumes
         └───────────────────┘
           Account updated event

  Infrastructure services :
  ├── Eureka Server      ← service discovery
  ├── Config Server      ← configuration centralisée (Git)
  └── Auth Server        ← OAuth2 / JWT token issuer
```

---

## Account Service — gestion des comptes et transactions

```java
// Account entity avec MongoDB
@Document(collection = "accounts")
public class Account {

    @Id
    private String name;

    @NotNull
    @Size(min = 1, max = 20)
    private List<Item> incomes;

    @NotNull
    @Size(min = 1, max = 20)
    private List<Item> expenses;

    @NotNull
    @Size(min = 1, max = 10)
    private List<Saving> savings;

    private AccountNote note;

    @LastModifiedDate
    private Date lastSeen;
}

// Service — logique métier compte
@Service
public class AccountServiceImpl implements AccountService {

    @Autowired private AccountRepository  repository;
    @Autowired private StatisticsService  statisticsService;
    @Autowired private NotificationClient notificationClient;

    @Override
    public Account create(User user) {
        Account existing = repository.findById(user.getUsername()).orElse(null);
        if (existing != null) {
            throw new AlreadyExistsException(user.getUsername());
        }

        // Créer le compte avec des valeurs par défaut
        Account account = new Account();
        account.setLastSeen(new Date());
        account.setSavings(Collections.emptyList());
        account.setNote(new AccountNote());
        repository.save(account);

        // Configurer les notifications par défaut
        notificationClient.createAccount(user);

        log.info("New account created: {}", user.getUsername());
        return account;
    }

    @Override
    public void saveChanges(String name, Account update) {
        Account account = repository.findById(name)
            .orElseThrow(() -> new NotFoundException(name));

        account.setIncomes(update.getIncomes());
        account.setExpenses(update.getExpenses());
        account.setSavings(update.getSavings());
        account.setNote(update.getNote());
        account.setLastSeen(new Date());
        repository.save(account);

        // Notifier le Statistics Service de la mise à jour
        statisticsService.updateStatistics(name, account);
    }
}
```

---

## Config Server centralisé — une seule source de vérité

```yaml
# config-server/src/main/resources/application.yml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/mon-repo/piggymetrics-config
          # Chaque service lit sa config depuis ce dépôt Git
          # account-service.yml, statistics-service.yml, etc.
          # Avantage : changer une config sans redéployer le service

# account-service.yml (dans le dépôt config)
spring:
  data:
    mongodb:
      host: account-mongodb      # référence au service Docker
      port: 27017
      database: accounts

feign:
  hystrix:
    enabled: true    # circuit breaker sur les appels inter-services

notification:
  email:
    host: ${MAIL_HOST}           # variable d'environnement
    port: ${MAIL_PORT}
    username: ${MAIL_USERNAME}
    password: ${MAIL_PASSWORD}
```

---

## Communication inter-services avec Feign + Hystrix

```java
// Account Service appelle Statistics Service via Feign
@FeignClient(name = "statistics-service", fallback = StatisticsFallback.class)
public interface StatisticsServiceClient {

    @PutMapping("/statistics/{accountName}")
    void updateStatistics(
        @PathVariable("accountName") String accountName,
        @RequestBody Account account
    );
}

// Fallback : que faire si Statistics Service est down ?
@Component
public class StatisticsFallback implements StatisticsServiceClient {

    @Override
    public void updateStatistics(String accountName, Account account) {
        // Dégradation gracieuse : logger l'échec, ne pas faire échouer la requête principale
        log.error("Statistics service is unavailable. "
                + "Skipping statistics update for account: {}", accountName);
        // L'utilisateur peut quand même sauvegarder son compte
        // Les statistiques seront mises à jour lors du prochain appel réussi
    }
}
```

---

## Service Discovery — Eureka en pratique

```java
// Chaque service s'enregistre automatiquement
@SpringBootApplication
@EnableDiscoveryClient   // ← enregistrement Eureka automatique
public class AccountServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(AccountServiceApplication.class, args);
    }
}

// L'API Gateway résout les noms de services dynamiquement
// Pas besoin de hardcoder les URLs — Eureka gère la localisation
// "statistics-service" → Eureka → http://10.0.0.5:8082
```

---

## Tests — AccountService isolé

```java
@ExtendWith(MockitoExtension.class)
class AccountServiceTest {

    @Mock AccountRepository    repository;
    @Mock StatisticsService    statisticsService;
    @Mock NotificationClient   notificationClient;

    @InjectMocks AccountServiceImpl accountService;

    @Test
    void create_shouldSaveAccount_whenUsernameNotTaken() {
        User user = new User();
        user.setUsername("testuser");

        when(repository.findById("testuser")).thenReturn(Optional.empty());

        Account created = accountService.create(user);

        verify(repository, times(1)).save(any(Account.class));
        verify(notificationClient, times(1)).createAccount(user);
        assertNotNull(created.getLastSeen());
    }

    @Test
    void create_shouldThrowException_whenUsernameAlreadyExists() {
        when(repository.findById(any())).thenReturn(Optional.of(new Account()));

        assertThrows(AlreadyExistsException.class,
            () -> accountService.create(new User()));
    }
}
```

---

## Ce que j'ai appris

Le **circuit breaker** (Hystrix/Resilience4j) est la leçon la plus importante
des microservices. Sans lui, si Statistics Service est down, l'appel Feign
bloque pendant 30 secondes avant de timeout, monopolise un thread, et finit
par faire tomber Account Service en cascade — alors que Account Service
n'a aucun problème.

Avec le fallback, Account Service continue à fonctionner normalement.
Les statistiques sont momentanément en retard — acceptable. L'utilisateur
ne voit aucune erreur. C'est ça la résilience distribuée.

---

*Projet réalisé dans le cadre de ma formation ingénieur — ENSET Mohammedia*
*Par **Abderrahmane Elouafi** · [LinkedIn](https://www.linkedin.com/in/abderrahmane-elouafi-43226736b/) · [Portfolio](https://my-first-porfolio-six.vercel.app/)*
