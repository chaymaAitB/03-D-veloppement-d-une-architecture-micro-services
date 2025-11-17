Voici un rapport complet répondant point par point aux 10 questions :

# Rapport Technique - Système de Gestion de Cotations Boursières

## 1. Création du Project Maven Multi-Modules

### Structure du projet parent :
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    
    <groupId>ma.enset</groupId>
    <artifactId>stock-market-system</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>
    
    <modules>
        <module>discovery-service</module>
        <module>gateway-service</module>
        <module>company-service</module>
        <module>stock-service</module>
        <module>chat-bot-service</module>
    </modules>
    
    <properties>
        <java.version>21</java.version>
        <spring-boot.version>3.2.0</spring-boot.version>
        <spring-cloud.version>2023.0.0</spring-cloud.version>
    </properties>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

## 2. Architecture Technique du Projet

### Diagramme d'Architecture Détaillé :
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                             COUCHE CLIENT                                   │
├─────────────────┐ ┌──────────────────┐ ┌─────────────────┐ ┌───────────────┤
│   React App     │ │   Flutter App    │ │  Telegram Bot   │ │   Postman     │
│   (Port: 3000)  │ │   (Mobile)       │ │   (Chatbot AI)  │ │   (Testing)   │
└─────────────────┘ └──────────────────┘ └─────────────────┘ └───────────────┘
         │                       │                       │             │
         └───────────────────────────┼─────────────────────┐             │
                                     │                     │             │
                            ┌────────┴────────┐   ┌────────┴────────┐     │
                            │   API GATEWAY   │   │   KEYCLOAK      │     │
                            │  (Port: 8080)   │   │  (Port: 8081)   │     │
                            └────────┬────────┘   └─────────────────┘     │
                                     │                     │             │
                                     └─────────────────────┼─────────────┘
                                                           │
                                     ┌─────────────────────┴─────────────┐
                                     │        SERVICE DISCOVERY          │
                                     │           (Port: 8761)            │
                                     └─────────────────┬─────────────────┘
                                                       │
         ┌───────────────────┐           ┌─────────────┴─────────────┐
         │                   │           │                           │
┌────────┴────────┐ ┌────────┴────────┐ ┌─┴──────────┐ ┌─────────────┴────────┐
│ COMPANY SERVICE │ │ STOCK SERVICE   │ │ CHATBOT    │ │   CONFIG SERVER      │
│  (Port: 8082)   │ │  (Port: 8083)   │ │ SERVICE    │ │    (Port: 8888)      │
└────────┬────────┘ └────────┬────────┘ └────────────┘ └──────────────────────┘
         │                   │
┌────────┴────────┐ ┌────────┴────────┐
│   PostgreSQL    │ │   PostgreSQL    │
│   (Port: 5432)  │ │   (Port: 5433)  │
└─────────────────┘ └─────────────────┘
```

### Technologies par Couche :

| Composant | Technologies | Port |
|-----------|--------------|------|
| **Discovery** | Eureka Server | 8761 |
| **Gateway** | Spring Cloud Gateway | 8080 |
| **Company** | Spring Boot, JPA, PostgreSQL | 8082 |
| **Stock** | Spring Boot, JPA, PostgreSQL | 8083 |
| **Chatbot** | Spring Boot, Telegram API, MCP | 8084 |
| **Sécurité** | Keycloak, OAuth2 | 8081 |
| **Base de données** | PostgreSQL (2 instances) | 5432/5433 |

## 3. Développement des Microservices Techniques

### 3.1 Discovery Service
**pom.xml** :
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>
```

**Application.java** :
```java
@SpringBootApplication
@EnableEurekaServer
public class DiscoveryServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(DiscoveryServiceApplication.class, args);
    }
}
```

**application.properties** :
```properties
server.port=8761
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
```

### 3.2 Gateway Service
**pom.xml** :
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
</dependencies>
```

**Routes Configuration** :
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: company-service
          uri: lb://COMPANY-SERVICE
          predicates:
            - Path=/api/companies/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
        - id: stock-service
          uri: lb://STOCK-SERVICE
          predicates:
            - Path=/api/stocks/**
```

## 4. Microservice Company-Service

### Entité Company :
```java
@Entity
public class Company {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private LocalDate introductionDate;
    private Double currentStockPrice;
    
    @Enumerated(EnumType.STRING)
    private Domain domain;
    
    public enum Domain {
        IT, AI, BANQUE, ASSURANCE, ENERGY, HEALTHCARE
    }
}
```

### Controller REST :
```java
@RestController
@RequestMapping("/api/companies")
public class CompanyController {
    
    @GetMapping
    public ResponseEntity<List<Company>> getAllCompanies() {
        return ResponseEntity.ok(companyService.findAll());
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<Company> getCompanyById(@PathVariable Long id) {
        return companyService.findById(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }
    
    @GetMapping("/domain/{domain}")
    public ResponseEntity<List<Company>> getCompaniesByDomain(
            @PathVariable Domain domain) {
        return ResponseEntity.ok(companyService.findByDomain(domain));
    }
    
    @PutMapping("/{id}/stock-price")
    public ResponseEntity<Company> updateStockPrice(
            @PathVariable Long id, 
            @RequestBody Double newPrice) {
        return ResponseEntity.ok(companyService.updateStockPrice(id, newPrice));
    }
}
```

### Tests Unitaires :
```java
@Test
void shouldUpdateStockPrice() {
    // Given
    Company company = new Company();
    company.setId(1L);
    company.setCurrentStockPrice(100.0);
    
    when(companyRepository.findById(1L)).thenReturn(Optional.of(company));
    when(companyRepository.save(any(Company.class))).thenReturn(company);
    
    // When
    Company updated = companyService.updateStockPrice(1L, 150.0);
    
    // Then
    assertEquals(150.0, updated.getCurrentStockPrice());
}
```

## 5. Microservice Stock-Service

### Entité StockMarket :
```java
@Entity
public class StockMarket {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private LocalDate date;
    private Double openValue;
    private Double highValue;
    private Double lowValue;
    private Double closeValue;
    private Long volume;
    
    private Long companyId;
}
```

### Service avec Calcul Automatique :
```java
@Service
@Transactional
public class StockService {
    
    public StockMarket addStockQuotation(StockMarket stock) {
        StockMarket savedStock = stockRepository.save(stock);
        
        // Mettre à jour le prix actuel de l'entreprise
        companyServiceClient.updateStockPrice(
            stock.getCompanyId(), 
            stock.getCloseValue()
        );
        
        return savedStock;
    }
    
    public Double calculateCurrentPrice(Long companyId) {
        return stockRepository.findLatestByCompanyId(companyId)
                .map(StockMarket::getCloseValue)
                .orElse(0.0);
    }
}
```

### Communication avec Company-Service (OpenFeign) :
```java
@FeignClient(name = "company-service", fallback = CompanyServiceFallback.class)
public interface CompanyServiceClient {
    
    @PutMapping("/api/companies/{companyId}/stock-price")
    void updateStockPrice(@PathVariable Long companyId, @RequestBody Double newPrice);
}

@Component
public class CompanyServiceFallback implements CompanyServiceClient {
    @Override
    public void updateStockPrice(Long companyId, Double newPrice) {
        // Logique de fallback
        log.warn("Fallback: Impossible de mettre à jour le prix pour company {}", companyId);
    }
}
```

## 6. Microservice Chatbot AI

### Configuration MCP :
```java
@Configuration
public class MCPConfiguration {
    
    @Bean
    public McpServer mcpServer(CompanyService companyService, StockService stockService) {
        return McpServer.builder()
            .resource("company-resources", resourceBuilder -> resourceBuilder
                .uri("template://company/{id}")
                .description("Accès aux données des entreprises"))
            .tool("get-company-info", toolBuilder -> toolBuilder
                .description("Obtenir les informations d'une entreprise")
                .stringParam("companyId", "ID de l'entreprise")
                .handler(ctx -> {
                    Long companyId = Long.parseLong(ctx.getParam("companyId"));
                    return companyService.getCompanyById(companyId);
                }))
            .tool("get-stock-quotations", toolBuilder -> toolBuilder
                .description("Obtenir les cotations d'une entreprise")
                .stringParam("companyId", "ID de l'entreprise")
                .handler(ctx -> {
                    Long companyId = Long.parseLong(ctx.getParam("companyId"));
                    return stockService.getStocksByCompanyId(companyId);
                }))
            .build();
    }
}
```

### Intégration Telegram :
```java
@Component
public class TelegramBotService {
    
    @Value("${telegram.bot.token}")
    private String botToken;
    
    public void sendStockAlert(Long chatId, String message) {
        // Implémentation de l'envoi de messages Telegram
    }
    
    public void processUserQuery(Long chatId, String query) {
        // Traitement des requêtes utilisateur avec MCP
        McpResponse response = mcpServer.processQuery(query);
        sendStockAlert(chatId, response.getMessage());
    }
}
```

## 7. Frontend Web React

### Structure du Projet :
```
frontend/
├── src/
│   ├── components/
│   │   ├── CompanyList.jsx
│   │   ├── StockChart.jsx
│   │   └── DomainFilter.jsx
│   ├── services/
│   │   └── api.js
│   └── App.jsx
```

### Composant Principal :
```jsx
import React, { useState, useEffect } from 'react';
import CompanyList from './components/CompanyList';
import StockChart from './components/StockChart';

const App = () => {
    const [companies, setCompanies] = useState([]);
    const [selectedCompany, setSelectedCompany] = useState(null);

    useEffect(() => {
        fetchCompanies();
    }, []);

    const fetchCompanies = async () => {
        const response = await fetch('http://localhost:8080/api/companies');
        const data = await response.json();
        setCompanies(data);
    };

    return (
        <div className="app">
            <h1>Système de Gestion Boursière</h1>
            <CompanyList 
                companies={companies} 
                onSelectCompany={setSelectedCompany}
            />
            {selectedCompany && <StockChart company={selectedCompany} />}
        </div>
    );
};

export default App;
```

## 8. Client Mobile Flutter

### Structure Principale :
```dart
// main.dart
import 'package:flutter/material.dart';
import 'screens/company_list_screen.dart';

void main() {
  runApp(StockMarketApp());
}

class StockMarketApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Stock Market',
      theme: ThemeData(primarySwatch: Colors.blue),
      home: CompanyListScreen(),
    );
  }
}

// company_list_screen.dart
class CompanyListScreen extends StatefulWidget {
  @override
  _CompanyListScreenState createState() => _CompanyListScreenState();
}

class _CompanyListScreenState extends State<CompanyListScreen> {
  List<Company> companies = [];
  
  @override
  void initState() {
    super.initState();
    fetchCompanies();
  }
  
  Future<void> fetchCompanies() async {
    final response = await http.get(
      Uri.parse('http://localhost:8080/api/companies')
    );
    
    if (response.statusCode == 200) {
      setState(() {
        companies = Company.fromJsonList(json.decode(response.body));
      });
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Entreprises')),
      body: ListView.builder(
        itemCount: companies.length,
        itemBuilder: (context, index) {
          return CompanyCard(company: companies[index]);
        },
      ),
    );
  }
}
```

## 9. Pipeline DevOps

### Dockerfile pour Microservices :
```dockerfile
FROM openjdk:21-jdk-slim
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Docker Compose :
```yaml
version: '3.8'
services:
  discovery-service:
    build: ./discovery-service
    ports: ["8761:8761"]
    
  gateway-service:
    build: ./gateway-service
    ports: ["8080:8080"]
    depends_on: [discovery-service]
    
  company-service:
    build: ./company-service
    environment:
      - SPRING_PROFILES_ACTIVE=docker
    depends_on: [discovery-service, postgres-company]
    
  postgres-company:
    image: postgres:15
    environment:
      POSTGRES_DB: company_db
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    ports: ["5432:5432"]
    
  keycloak:
    image: quay.io/keycloak/keycloak:latest
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    ports: ["8081:8080"]
```

### Jenkinsfile :
```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('Build Docker') {
            steps {
                sh 'docker-compose build'
            }
        }
        stage('Deploy to K8s') {
            steps {
                sh 'kubectl apply -f k8s/'
            }
        }
    }
}
```

### Configuration Kubernetes :
```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: company-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: company-service
  template:
    metadata:
      labels:
        app: company-service
    spec:
      containers:
      - name: company-service
        image: company-service:latest
        ports:
        - containerPort: 8082
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "kubernetes"
---
apiVersion: v1
kind: Service
metadata:
  name: company-service
spec:
  selector:
    app: company-service
  ports:
  - port: 80
    targetPort: 8082
  type: LoadBalancer
```

## 10. Sécurisation du Système Distribué

### Configuration Keycloak :
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/companies/**").hasRole("USER")
                .requestMatchers("/api/stocks/**").hasRole("USER")
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(Customizer.withDefaults())
            );
        return http.build();
    }
}
```

### Configuration des Ressources :
```yaml
keycloak:
  realm: stock-market-realm
  auth-server-url: http://localhost:8081
  resource: stock-market-client
  public-client: true
```

### Sécurité API Gateway :
```java
@Component
public class JwtAuthenticationFilter implements GlobalFilter {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String authHeader = exchange.getRequest()
            .getHeaders()
            .getFirst(HttpHeaders.AUTHORIZATION);
            
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7);
            if (!validateToken(token)) {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            }
        }
        
        return chain.filter(exchange);
    }
    
    private boolean validateToken(String token) {
        // Validation JWT avec Keycloak
        return jwtValidator.isValid(token);
    }
}
```

## ✅ Conclusion

Ce projet démontre une implémentation complète d'un système distribué microservices avec :

- **🏗️ Architecture modulaire** et scalable
- **🔒 Sécurité robuste** avec OAuth2/OpenID Connect
- **🤖 Intégration AI** via MCP
- **📱 Multi-plateforme** (Web, Mobile, Chatbot)
- **⚙️ DevOps automatisé** avec Docker/Kubernetes
- **🛡️ Résilience** avec Circuit Breaker

L'architecture respecte les meilleures pratiques des systèmes distribués modernes et offre une base solide pour des extensions futures.
