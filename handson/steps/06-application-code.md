# Part 6 — The application code

[← Part 5 — Jumpbox and GitHub self-hosted runner](05-jumpbox-and-runner.md) | [Index](../README.md) | [Part 7 — Multistage Dockerfiles and local scans →](07-dockerfiles-and-local-scans.md)

---

## 6.1 Repository root files

`.gitignore`:

```text
# Build output
node_modules/
dist/
target/

# Terraform
.terraform/
*.tfstate
*.tfstate.*
tfplan
plan.txt
crash.log

# Local files
.kubeconfig
*.pfx
*.key
*.crt
.env
```

`.trivyignore` (why this exception is fine is explained in Part 7.5):

```text
# App Gateway is the public entry point: it must accept 80/443 from the internet. WAF policy protects it.
AZU-0047
```

## 6.2 Backend (Spring Boot 4, Java 21)

Generate the skeleton (this gives you `mvnw`, the folder structure and `BackendApplication.java`):

```bash
cd ~/three-tier-aks
curl -sf https://start.spring.io/starter.zip \
  -d type=maven-project -d language=java -d bootVersion=4.1.2 -d javaVersion=21 \
  -d groupId=com.example -d artifactId=backend -d name=backend -d packageName=com.example.backend \
  -d dependencies=web,data-jpa,postgresql,flyway,actuator,prometheus,validation \
  -o backend.zip && unzip -o backend.zip -d backend && rm backend.zip
rm -f backend/HELP.md backend/src/main/resources/application.properties \
      backend/src/test/java/com/example/backend/BackendApplicationTests.java
```

> If start.spring.io says 4.1.2 is no longer offered, remove `-d bootVersion=4.1.2` and then set `<version>` in the pom below to the version it generated.
>
> We delete the generated `BackendApplicationTests` because it starts the whole app and needs a real database. CI has no database, so we use a web-layer test instead (6.2.8).

**Spring Boot 4 note:** starters are now modular. You'll see `spring-boot-starter-webmvc` (not `-web`), a separate `spring-boot-starter-flyway`, and matching `-test` starters. Older blog posts show the Boot 3 names.

### 6.2.1 `backend/pom.xml` (replace the generated one; it adds JaCoCo for code coverage)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>4.1.2</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.example</groupId>
	<artifactId>backend</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>backend</name>
	<description/>
	<url/>
	<licenses>
		<license/>
	</licenses>
	<developers>
		<developer/>
	</developers>
	<scm>
		<connection/>
		<developerConnection/>
		<tag/>
		<url/>
	</scm>
	<properties>
		<java.version>21</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-flyway</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-validation</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-webmvc</artifactId>
		</dependency>
		<dependency>
			<groupId>org.flywaydb</groupId>
			<artifactId>flyway-database-postgresql</artifactId>
		</dependency>

		<dependency>
			<groupId>io.micrometer</groupId>
			<artifactId>micrometer-registry-prometheus</artifactId>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>org.postgresql</groupId>
			<artifactId>postgresql</artifactId>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-flyway-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-validation-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-webmvc-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
			<!-- Code coverage report for SonarQube Cloud -->
			<plugin>
				<groupId>org.jacoco</groupId>
				<artifactId>jacoco-maven-plugin</artifactId>
				<version>0.8.16</version>
				<executions>
					<execution>
						<goals>
							<goal>prepare-agent</goal>
						</goals>
					</execution>
					<execution>
						<id>report</id>
						<phase>verify</phase>
						<goals>
							<goal>report</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>

</project>
```

### 6.2.2 `backend/src/main/resources/application.yml`

```yaml
spring:
  application:
    name: backend
  datasource:
    # In AKS these come from Key Vault (via the CSI driver -> Kubernetes Secret -> env vars).
    # The defaults are for running locally against docker-compose.
    url: ${DB_URL:jdbc:postgresql://localhost:5432/appdb}
    username: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:postgres}
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate # Flyway owns the schema
  flyway:
    enabled: true
  lifecycle:
    timeout-per-shutdown-phase: 20s

server:
  port: 8080
  shutdown: graceful # finish in-flight requests when Kubernetes sends SIGTERM

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      probes:
        enabled: true # /actuator/health/liveness and /actuator/health/readiness
      group:
        readiness:
          include: readinessState,db # not ready if the database is unreachable
  metrics:
    tags:
      application: ${spring.application.name}
    distribution:
      percentiles-histogram:
        http.server.requests: true # enables p95/p99 latency queries in Prometheus
```

What each part gives you:

- `${DB_URL:...}`: the env var if set, otherwise the default. The same image runs locally and in AKS (12-factor config).
- `ddl-auto: validate` + **Flyway**: the schema is versioned in SQL files (`V1__...sql`). Hibernate only checks it, so it can never drop a column by accident. This is your answer to "how do you handle database changes in CI/CD and rollback?"
- `server.shutdown: graceful`: on SIGTERM, Spring stops taking new requests and finishes the in-flight ones (zero-downtime deploys).
- **Actuator probes**: `/actuator/health/liveness` (is the JVM alive?) and `/actuator/health/readiness` (can it serve traffic?). Readiness includes `db`, so if PostgreSQL is unreachable the pod becomes NotReady and gets **no traffic**, but it is **not restarted**.
- `/actuator/prometheus`: metrics for Prometheus. `percentiles-histogram` enables p95/p99 latency.

### 6.2.3 Database migration — `backend/src/main/resources/db/migration/V1__create_todo.sql`

```sql
CREATE TABLE todo (
    id         BIGSERIAL PRIMARY KEY,
    title      VARCHAR(200) NOT NULL,
    done       BOOLEAN      NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ  NOT NULL DEFAULT now()
);
```

### 6.2.4 `backend/src/main/java/com/example/backend/BackendApplication.java` (generated, keep it)

```java
package com.example.backend;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class BackendApplication {

	public static void main(String[] args) {
		SpringApplication.run(BackendApplication.class, args);
	}

}
```

### 6.2.5 Entity — `Todo.java`

```java
package com.example.backend;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import java.time.Instant;

@Entity
public class Todo {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 200)
    private String title;

    @Column(nullable = false)
    private boolean done;

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt = Instant.now();

    protected Todo() {
    }

    public Todo(String title) {
        this.title = title;
    }

    public Long getId() { return id; }
    public String getTitle() { return title; }
    public boolean isDone() { return done; }
    public Instant getCreatedAt() { return createdAt; }
    public void setDone(boolean done) { this.done = done; }
}
```

### 6.2.6 Repository — `TodoRepository.java`

```java
package com.example.backend;

import java.util.List;
import org.springframework.data.jpa.repository.JpaRepository;

public interface TodoRepository extends JpaRepository<Todo, Long> {
    List<Todo> findAllByOrderByIdAsc();
}
```

### 6.2.7 REST controller — `TodoController.java`

```java
package com.example.backend;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import java.net.InetAddress;
import java.net.UnknownHostException;
import java.util.List;
import java.util.Map;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.server.ResponseStatusException;

@RestController
@RequestMapping("/api")
public class TodoController {

    public record NewTodo(@NotBlank @Size(max = 200) String title) {
    }

    private final TodoRepository repository;

    public TodoController(TodoRepository repository) {
        this.repository = repository;
    }

    @GetMapping("/todos")
    public List<Todo> list() {
        return repository.findAllByOrderByIdAsc();
    }

    @PostMapping("/todos")
    @ResponseStatus(HttpStatus.CREATED)
    public Todo create(@Valid @RequestBody NewTodo body) {
        return repository.save(new Todo(body.title()));
    }

    @PutMapping("/todos/{id}/toggle")
    public Todo toggle(@PathVariable Long id) {
        Todo todo = repository.findById(id)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
        todo.setDone(!todo.isDone());
        return repository.save(todo);
    }

    @DeleteMapping("/todos/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        repository.deleteById(id);
    }

    // Shows which pod answered: useful to see load balancing across replicas.
    @GetMapping("/info")
    public Map<String, String> info() throws UnknownHostException {
        return Map.of("pod", InetAddress.getLocalHost().getHostName(), "version",
                System.getenv().getOrDefault("APP_VERSION", "local"));
    }
}
```

### 6.2.8 Unit test — `backend/src/test/java/com/example/backend/TodoControllerTest.java`

```java
package com.example.backend;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

import java.util.List;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;

// Tests only the web layer, so CI does not need a database.
@WebMvcTest(TodoController.class)
class TodoControllerTest {

    @Autowired
    private MockMvc mvc;

    @MockitoBean
    private TodoRepository repository;

    @Test
    void listsTodos() throws Exception {
        when(repository.findAllByOrderByIdAsc()).thenReturn(List.of(new Todo("learn AKS")));

        mvc.perform(get("/api/todos"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$[0].title").value("learn AKS"));
    }

    @Test
    void createsTodo() throws Exception {
        when(repository.save(any(Todo.class))).thenAnswer(inv -> inv.getArgument(0));

        mvc.perform(post("/api/todos").contentType(MediaType.APPLICATION_JSON)
                        .content("{\"title\":\"write helm chart\"}"))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.title").value("write helm chart"));
    }

    @Test
    void rejectsBlankTitle() throws Exception {
        mvc.perform(post("/api/todos").contentType(MediaType.APPLICATION_JSON)
                        .content("{\"title\":\"\"}"))
                .andExpect(status().isBadRequest());
    }
}
```

`@WebMvcTest` starts only the web layer, and `@MockitoBean` replaces the repository. These tests need no database, so they run in seconds in CI. (`@MockBean` was removed in Boot 4; `@MockitoBean` replaces it.)

Run the tests without installing Java:

```bash
cd ~/three-tier-aks/backend
docker run --rm -v "$PWD":/w -v "$HOME/.m2":/root/.m2 -w /w maven:3.9-eclipse-temurin-21 mvn -B verify
# BUILD SUCCESS, Tests run: 3, and target/site/jacoco/jacoco.xml exists
```

## 6.3 Frontend (React + Vite, served by nginx)

The frontend is small on purpose. The lab is about the platform, not React.

`frontend/package.json`:

```json
{
  "name": "frontend",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "19.3.1",
    "react-dom": "19.3.1"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "6.1.2",
    "vite": "8.3.5"
  }
}
```

`frontend/vite.config.js`:

```javascript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    // Local development only: forward /api to the Spring Boot app.
    proxy: { '/api': 'http://localhost:8080' },
  },
})
```

`frontend/index.html`:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Three-tier Todo</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

`frontend/src/main.jsx`:

```jsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import App from './App.jsx'

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

`frontend/src/App.jsx`:

```jsx
import { useEffect, useState } from 'react'

// The browser only ever calls /api on the same host.
// nginx (in the frontend pod) forwards /api to the backend Service inside the cluster.
export default function App() {
  const [todos, setTodos] = useState([])
  const [title, setTitle] = useState('')
  const [info, setInfo] = useState(null)
  const [error, setError] = useState('')

  async function load() {
    try {
      const [todosRes, infoRes] = await Promise.all([fetch('/api/todos'), fetch('/api/info')])
      if (!todosRes.ok) throw new Error(`GET /api/todos -> ${todosRes.status}`)
      setTodos(await todosRes.json())
      setInfo(await infoRes.json())
      setError('')
    } catch (e) {
      setError(e.message)
    }
  }

  useEffect(() => {
    load()
  }, [])

  async function add(e) {
    e.preventDefault()
    if (!title.trim()) return
    await fetch('/api/todos', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ title }),
    })
    setTitle('')
    load()
  }

  async function toggle(id) {
    await fetch(`/api/todos/${id}/toggle`, { method: 'PUT' })
    load()
  }

  async function remove(id) {
    await fetch(`/api/todos/${id}`, { method: 'DELETE' })
    load()
  }

  return (
    <main style={{ fontFamily: 'sans-serif', maxWidth: 560, margin: '40px auto', padding: '0 16px' }}>
      <h1>Three-tier Todo</h1>
      <p>React (nginx) → Spring Boot → PostgreSQL on private AKS</p>
      {info && <p><small>Answered by pod <b>{info.pod}</b>, version <b>{info.version}</b></small></p>}
      {error && <p style={{ color: 'crimson' }}>Error: {error}</p>}

      <form onSubmit={add} style={{ display: 'flex', gap: 8 }}>
        <input value={title} onChange={(e) => setTitle(e.target.value)} placeholder="New todo" style={{ flex: 1 }} />
        <button type="submit">Add</button>
      </form>

      <ul>
        {todos.map((t) => (
          <li key={t.id}>
            <label style={{ textDecoration: t.done ? 'line-through' : 'none' }}>
              <input type="checkbox" checked={t.done} onChange={() => toggle(t.id)} /> {t.title}
            </label>{' '}
            <button onClick={() => remove(t.id)}>x</button>
          </li>
        ))}
      </ul>
    </main>
  )
}
```

`frontend/nginx/default.conf.template`:

```nginx
# ${BACKEND_URL} is replaced at container start by the nginx image's envsubst step.
server {
    listen 8080;
    root /usr/share/nginx/html;

    # Used by Kubernetes probes AND by Application Gateway's health probe.
    location = /healthz {
        access_log off;
        default_type text/plain;
        return 200 "ok\n";
    }

    # Reverse proxy: browser -> /api/... -> backend Service (ClusterIP) inside the cluster.
    location /api/ {
        proxy_pass ${BACKEND_URL};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # React single-page app: unknown paths fall back to index.html.
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

**How the frontend connects to the backend (an interview question).** The browser only calls `/api/...` on the **same host**, so there is no CORS and no backend URL baked into the JavaScript. nginx in the frontend pod forwards `/api/` to the backend **ClusterIP Service** (`BACKEND_URL`), which is never exposed to the internet. The nginx image runs `envsubst` on `/etc/nginx/templates/*.template` at start-up and replaces `${BACKEND_URL}` with the environment variable. The same image works locally (docker-compose) and in AKS (Helm sets the variable).

Create the lock file (CI uses `npm ci`, which **requires** `package-lock.json`):

```bash
cd ~/three-tier-aks/frontend
npm install            # or: docker run --rm -v "$PWD":/w -w /w node:24-alpine npm install
npm run build          # creates dist/
```

Commit `package-lock.json`. It pins every transitive dependency, so builds are reproducible and Trivy can scan exact versions.

## 6.4 Run all three tiers locally

docker-compose builds the images with the Dockerfiles from Part 7. **Create `backend/Dockerfile`, `backend/.dockerignore`, `frontend/Dockerfile` and `frontend/.dockerignore` from [7.1](07-dockerfiles-and-local-scans.md#71-backend--backenddockerfile) and [7.2](07-dockerfiles-and-local-scans.md#72-frontend--frontenddockerfile) first**, then come back here.

`docker-compose.yml` at the repo root:

```yaml
# Local run of all three tiers: http://localhost:3000
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d appdb"]
      interval: 5s
      retries: 10

  backend:
    build: ./backend
    environment:
      DB_URL: jdbc:postgresql://db:5432/appdb
      DB_USERNAME: postgres
      DB_PASSWORD: postgres
    depends_on:
      db:
        condition: service_healthy

  frontend:
    build: ./frontend
    environment:
      BACKEND_URL: http://backend:8080
    ports:
      - "3000:8080"
    depends_on:
      - backend
```

```bash
cd ~/three-tier-aks
docker compose up -d --build
curl -s -X POST -H 'Content-Type: application/json' -d '{"title":"first todo"}' localhost:3000/api/todos
curl -s localhost:3000/api/todos
curl -s localhost:3000/api/info
# open http://localhost:3000 in the browser
docker compose logs backend | grep -i flyway   # "Successfully applied 1 migration"
docker compose down -v
```

When this works locally, you know any problem in AKS is about **the platform** (network, identity, config), not the code. Say this in interviews: "first reproduce locally, then isolate the layer."

```bash
git add . && git commit -m "Three-tier app: React, Spring Boot, Flyway, docker-compose" && git push
```

---

[← Part 5 — Jumpbox and GitHub self-hosted runner](05-jumpbox-and-runner.md) | [Index](../README.md) | [Part 7 — Multistage Dockerfiles and local scans →](07-dockerfiles-and-local-scans.md)
