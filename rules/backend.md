# Go Backend Project Guidelines

A reference guide for structuring, building, and deploying Go backend projects using Echo, GORM, Goose, and related
tooling.

---

## Table of Contents

1. [Project Structure](#project-structure)
2. [Layer Dependencies](#layer-dependencies)
3. [Aggregate Pattern](#aggregate-pattern)
4. [Controllers](#controllers)
5. [Services](#services)
6. [Repositories](#repositories)
7. [DTOs](#dtos)
8. [Models](#models)
9. [Migrations](#migrations)
10. [Error Handling](#error-handling)
11. [Configuration](#configuration)
12. [Main Entry Point](#main-entry-point)
13. [Docker & Docker Compose](#docker--docker-compose)
14. [Dockerfile](#dockerfile)
15. [Environment Variables](#environment-variables)
16. [Routing Conventions](#routing-conventions)
17. [Authentication](#authentication)
18. [Input Validation](#input-validation)
19. [Multitenancy](#multitenancy)
20. [Frontend Static Hosting](#frontend-static-hosting)
21. [Kubernetes](#kubernetes)
22. [Swagger / OpenAPI Documentation](#swagger--openapi-documentation)
23. [Git & Tooling](#git--tooling)

---

## Project Structure

```
cmd/
  main.go              # Application entry point
internal/
  controller/          # HTTP handlers
  service/             # Business logic
  repository/          # Database access layer
  model/               # Data models (GORM structs)
  client/              # External service integrations (APIs, etc.)
  dto/                 # Data Transfer Objects (requests, responses, config)
  util/                # Utility functions and helpers
migrations/            # Goose SQL migration files
migrations.go          # Migration runner (package backend, in root)
docker-compose.yml
Dockerfile
.env
.env.example
.gitignore
build.sh               # (optional) Frontend build + deploy script
```

**File naming convention:** Use short, one-word filenames. For example, `service/user.go` for `UserService`,
`repository/article.go` for `ArticleRepository`.

---

## Layer Dependencies

Dependencies only flow in one direction. No layer may import a layer above it.

```
controller  →  service
service     →  repository
service     →  client
service     →  service   (internal cross-service calls are allowed)
```

---

## Aggregate Pattern

Each of the following packages must contain an aggregate file that exposes all sub-components through a single struct:
`controller/`, `service/`, `client/`, `repository/`.

### Repository Aggregate (`repository/repositories.go`)

```go
package repository

import "gorm.io/gorm"

type Repositories interface {
	User() UserRepository
}

type repositories struct {
	userRepository UserRepository
}

func NewRepositories(db *gorm.DB) Repositories {
	return &repositories{
		userRepository: newUserRepository(db),
	}
}

func (r *repositories) User() UserRepository {
	return r.userRepository
}
```

### Service Aggregate (`service/services.go`)

```go
package service

type Services interface {
	User() UserService
}

type services struct {
	userService UserService
}

func NewServices(repositories repository.Repositories, config dto.Config) Services {
	userService := newUserService(repositories.User(), config)
	return &services{
		userService: userService,
	}
}

func (s *services) User() UserService {
	return s.userService
}
```

### Controller Aggregate (`controller/controllers.go`)

The controller aggregate must also expose a `Route` method that registers all routes with the Echo router. All routes
must be prefixed with `/api/`. Do not use API groups.

```go
package controller

import (
	"github.com/labstack/echo/v5"
	"yourmodule/internal/service"
)

type Controllers interface {
	Route(e *echo.Echo)
}

type controllers struct {
	userController UserController
}

func NewControllers(services service.Services) Controllers {
	return &controllers{
		userController: newUserController(services.User()),
	}
}

func (c controllers) Route(e *echo.Echo) {
	e.POST("/api/auth/login", c.userController.Login)
	// Register additional routes here
}
```

---

## Controllers

Controllers handle HTTP requests. They bind incoming payloads and delegate all logic to services.

**Rules:**

- Binding must happen inside the handler method, not in middleware.
- Use `*echo.Context` (crucial pointer!) as required by Echo v5.
- Call `userService.Authenticate(extractToken(c))` for protected routes. Do not add new middleware for authentication.
- Use `extractToken` helper (see below) to extract the Bearer token.

### Example Controller (`controller/user.go`)

```go
package controller

import (
	"net/http"

	"github.com/labstack/echo/v5"
	"yourmodule/internal/dto"
	"yourmodule/internal/service"
)

type UserController interface {
	Login(c *echo.Context) error
}

type userController struct {
	userService service.UserService
}

func newUserController(userService service.UserService) UserController {
	return &userController{
		userService: userService,
	}
}

func (u userController) Login(c *echo.Context) error {
	var loginRequest dto.LoginRequest
	if err := c.Bind(&loginRequest); err != nil {
		return err
	}
	token, err := u.userService.Login(loginRequest.Email, loginRequest.Password)
	if err != nil {
		return err
	}
	return c.JSON(http.StatusOK, dto.LoginResponse{Token: token})
}
```

### Example Resource Controller with Authentication (`controller/article.go`)

```go
func (a articleController) Index(c *echo.Context) error {
user, err := a.userService.Authenticate(extractToken(c))
if err != nil {
return err
}

articles, err := a.articleService.Index(c.Request().Context(), user, c.Param("workspaceID"), c.Param("collectionID"))
if err != nil {
return err
}
return c.JSON(http.StatusOK, articles)
}
```

### Token Extraction Helper (`controller/util.go`)

```go
package controller

import (
	"strings"

	"github.com/labstack/echo/v5"
)

func extractToken(c *echo.Context) string {
	return strings.ReplaceAll(c.Request().Header.Get(echo.HeaderAuthorization), "Bearer ", "")
}
```

### Error Handler Middleware (`controller/error_handler.go`)

```go
package controller

import (
	"errors"

	"github.com/labstack/echo/v5"
	"yourmodule/internal/dto"
)

var AppErrorHandler = func(next echo.HandlerFunc) echo.HandlerFunc {
	return func(c *echo.Context) error {
		err := next(c)
		if err != nil {
			var appError dto.AppError
			switch {
			case errors.As(err, &appError):
				return echo.NewHTTPError(400, err.Error())
			}
		}
		return err
	}
}
```

---

## Services

Services contain all business logic. They depend on repositories, clients, other services, and config — never on
controllers.

### Example Service (`service/user.go`)

```go
package service

import (
	"fmt"
	"strings"
	"time"

	"github.com/golang-jwt/jwt/v4"
	"golang.org/x/crypto/bcrypt"
	"yourmodule/internal/dto"
	"yourmodule/internal/model"
	"yourmodule/internal/repository"
)

type UserService interface {
	Login(email string, password string) (string, error)
	Authenticate(token string) (model.User, error)
}

type userService struct {
	userRepository repository.UserRepository
	config         dto.Config
}

func newUserService(userRepository repository.UserRepository, config dto.Config) UserService {
	return &userService{userRepository, config}
}

func (u userService) Login(email string, password string) (string, error) {
	email = strings.TrimSpace(strings.ToLower(email))
	user, err := u.userRepository.FindByEmail(email)
	if err != nil {
		return "", dto.AppError(err)
	}
	if user == nil {
		return "", dto.AppError(fmt.Errorf("user with email %s not found", email))
	}
	err = bcrypt.CompareHashAndPassword([]byte(user.Password), []byte(password))
	if err != nil {
		return "", dto.AppError(fmt.Errorf("invalid password"))
	}
	user.LastLoginAt = time.Now()
	err = u.userRepository.Update(user)
	if err != nil {
		return "", fmt.Errorf("error updating user")
	}
	return u.generateJwt(*user)
}

func (u userService) DecodeToken(token string) (string, error) {
	claims := jwt.MapClaims{}
	_, err := jwt.ParseWithClaims(token, claims, func(token *jwt.Token) (interface{}, error) {
		return []byte(u.config.SigningSecret), nil
	})
	if err != nil {
		return "", err
	}
	userID := claims["user_id"].(string)
	return userID, nil
}

func (u userService) Authenticate(token string) (model.User, error) {
	userID, err := u.DecodeToken(token)
	if err != nil {
		return model.User{}, err
	}
	user, err := u.userRepository.FindByID(userID)
	if err != nil {
		return model.User{}, err
	}
	if user == nil {
		return model.User{}, dto.AppError(fmt.Errorf("user with id %s not found", userID))
	}
	user.LastLoginAt = time.Now()
	err = u.userRepository.Update(user)
	if err != nil {
		return model.User{}, fmt.Errorf("error updating user")
	}
	return *user, nil
}
```

---

## Repositories

Repositories handle all database access via GORM. They must never contain business logic.

**Method signature rules:**

- `Store` takes a model value and returns a model value (no pointers).
- `Update` and `Delete` take and modify a pointer to a model.
- `FindBy*` methods return a pointer to the model, or `nil` if not found (no error on not found).

### Example Repository (`repository/user.go`)

```go
package repository

import (
	"errors"
	"strings"

	"gorm.io/gorm"
	"yourmodule/internal/model"
)

type UserRepository interface {
	Store(user model.User) (model.User, error)
	Update(user *model.User) error
	FindByID(id string) (*model.User, error)
	FindByEmail(email string) (*model.User, error)
}

type userRepository struct {
	db *gorm.DB
}

func newUserRepository(db *gorm.DB) UserRepository {
	return &user{db: db}
}

func (u userRepository) Store(user model.User) (model.User, error) {
	u.db.Create(&user)
	return user, nil
}

func (u userRepository) Update(user *model.User) error {
	return u.db.Save(user).Error
}

func (u userRepository) FindByID(id string) (*model.User, error) {
	var user model.User
	result := u.db.Where("id = ?", id).First(&user)
	if result.Error != nil {
		if errors.Is(result.Error, gorm.ErrRecordNotFound) {
			return nil, nil
		}
		return nil, result.Error
	}
	return &user, nil
}

func (u userRepository) FindByEmail(email string) (*model.User, error) {
	var user model.User
	result := u.db.Where("lower(email) = ?", strings.TrimSpace(strings.ToLower(email))).First(&user)
	if result.Error != nil {
		if errors.Is(result.Error, gorm.ErrRecordNotFound) {
			return nil, nil
		}
		return nil, result.Error
	}
	return &user, nil
}
```

---

## DTOs

DTOs live in `internal/dto/` and carry data between layers. Use them for HTTP request/response bodies and application
configuration. Keep every DTO in a separate file!

**File naming convention:**

- HTTP request bodies: `login_request.go` → struct `LoginRequest`
- HTTP response bodies: `login_response.go` → struct `LoginResponse`
- Application errors: `app_error.go`
- Configuration: `config.go`

### Application Config (`dto/config.go`)

```go
package dto

import "os"

type Config struct {
	DSN           string `json:"DSN"`
	SigningSecret string `json:"SIGNING_SECRET"`
}

func NewConfig() Config {
	return Config{
		DSN:           os.Getenv("DSN"),
		SigningSecret: os.Getenv("SIGNING_SECRET"),
	}
}
```

> When a new config variable is needed, also add it to `.env.example`.

### Application Error (`dto/app_error.go`)

```go
package dto

import "fmt"

type AppError error

var (
	NotFound       = AppError(fmt.Errorf("not found"))
	Unauthorized   = AppError(fmt.Errorf("unauthorized"))
	MailSendFailed = AppError(fmt.Errorf("failed to send email"))
)
```

---

## Models

GORM models live in `internal/model/`. Each file contains one model, named after the entity (e.g., `model/user.go`).

---

## Migrations

Use [Goose](https://github.com/pressly/goose) for database migrations. Store all `.sql` files in `migrations/`.

### Migration Runner (`migrations.go` — root package `backend`)

```go
package backend

import (
	"database/sql"
	"embed"

	"github.com/pressly/goose/v3"
	"github.com/sirupsen/logrus"
)

//go:embed migrations/*.sql
var embedMigrations embed.FS

func RunMigrations(db *sql.DB) {
	goose.SetBaseFS(embedMigrations)
	goose.SetLogger(logrus.StandardLogger())

	if err := goose.SetDialect("postgres"); err != nil {
		logrus.Panic(err)
	}

	if err := goose.Up(db, "migrations"); err != nil {
		logrus.Panic(err)
	}
	logrus.Info("Database migrations applied successfully")
}
```

### Goose Environment Variables

Set these when running Goose locally. Match `GOOSE_DBSTRING` to the values in your `docker-compose.yml`.

```env
GOOSE_DRIVER=postgres
GOOSE_MIGRATION_DIR=migrations
GOOSE_DBSTRING="host=localhost port=5889 user=postgres password=jqKwlS9vN0mfm1v dbname=postgres"
```

---

## Error Handling

- Use `dto.AppError` for domain-level errors (not found, unauthorized, etc.).
- The `AppErrorHandler` middleware in `controller/error_handler.go` maps `AppError` to HTTP 400 responses.
- Unexpected/internal errors pass through as-is and result in a 500.

---

## Configuration

- Load environment variables with `godotenv` at startup.
- Pass `dto.Config` (built from `dto.NewConfig()`) into services that need it.
- Always maintain `.env.example` with all supported variables.

---

## Main Entry Point (`cmd/main.go`)

```go
package main

import (
	"os"

	"github.com/joho/godotenv"
	"github.com/labstack/echo/v5"
	"github.com/sirupsen/logrus"
	"gorm.io/driver/postgres"
	"gorm.io/gorm"

	backend "yourmodule"
	"yourmodule/internal/controller"
	"yourmodule/internal/dto"
	"yourmodule/internal/repository"
	"yourmodule/internal/service"
)

func main() {
	logrus.SetFormatter(&logrus.TextFormatter{
		ForceColors: true,
	})

	err := godotenv.Load()
	if err != nil {
		logrus.Info("Error loading .env file")
	}

	db, err := gorm.Open(postgres.Open(os.Getenv("DSN")), &gorm.Config{})
	if err != nil {
		logrus.Panic(err)
	}

	sqlDB, err := db.DB()
	if err != nil {
		logrus.Panic(err)
	}
	backend.RunMigrations(sqlDB)

	config := dto.NewConfig()
	repositories := repository.NewRepositories(db)
	services := service.NewServices(repositories, config)
	controllers := controller.NewControllers(services)

	e := echo.New()
    e.Use(middleware.CORS("*"))
	controllers.Route(e)
	logrus.Info("starting server on port 3000")
	logrus.Fatal(e.Start(":3000"))
}
```

---

## Docker & Docker Compose

Randomize the following values on first generation and fill them into both `docker-compose.yml` and `.env` /
`.env.example`:

- Randomize Database password
- Randomize Database host port (avoid 5432)
- Randomize Application port (avoid 3000)
- Randomize JWT signing secret

### `docker-compose.yml`

```yaml
name: <project name>
services:
  db:
    image: postgres:18
    restart: always
    environment:
      POSTGRES_PASSWORD: "jqKwlS9vN0mfm1v"
    ports:
      - "5889:5432"
    volumes:
      - db:/var/lib/postgresql/18/docker
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U postgres -d postgres" ]
      interval: 10s
      timeout: 60s
      retries: 5

  app:
    build: .
    environment:
      DSN: host=db user=postgres password=jqKwlS9vN0mfm1v dbname=postgres sslmode=disable
    env_file:
      - .env
    restart: always
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "8471:3000"

volumes:
  db:
```

---

## Dockerfile

Multi-stage build to produce a minimal production image.

```dockerfile
FROM golang AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . ./
RUN GOOS=linux go build -o /main cmd/main.go

FROM golang
WORKDIR /
COPY --from=builder /main /main
EXPOSE 3000
ENTRYPOINT ["/main"]
```

---

## Environment Variables

### `.env.example`

```env
DSN=host=localhost port=5889 user=postgres password=jqKwlS9vN0mfm1v dbname=postgres sslmode=disable
SIGNING_SECRET=replace-with-random-secret

# Goose (local migrations only)
GOOSE_DRIVER=postgres
GOOSE_MIGRATION_DIR=migrations
GOOSE_DBSTRING=host=localhost port=5889 user=postgres password=jqKwlS9vN0mfm1v dbname=postgres
```

> Copy `.env.example` to `.env` and fill in real values. Never commit `.env` to version control.

---

## Routing Conventions

- All routes must be prefixed with `/api/`.
- Do not use Echo route groups.
- Use path parameters for nested resources.
- Register all routes in `controller/controllers.go` inside the `Route` method.

### Example Routes

```go
func (c controllers) Route(e *echo.Echo) {
e.POST("/api/auth/login", c.userController.Login)

e.GET("/api/workspaces/:workspaceID/collections", c.collectionController.Index)
e.POST("/api/workspaces/:workspaceID/collections", c.collectionController.Store)
e.PUT("/api/workspaces/:workspaceID/collections/:collectionID", c.collectionController.Update)
e.DELETE("/api/workspaces/:workspaceID/collections/:collectionID", c.collectionController.Destroy)

e.PUT("/api/workspaces/:workspaceID/collections/:collectionID/translations/:locale", c.collectionController.SetTranslation)
e.DELETE("/api/workspaces/:workspaceID/collections/:collectionID/translations/:locale", c.collectionController.DeleteTranslation)

e.GET("/api/workspaces/:workspaceID/collections/:collectionID/articles", c.articleController.Index)
e.POST("/api/workspaces/:workspaceID/collections/:collectionID/articles", c.articleController.Store)
e.GET("/api/workspaces/:workspaceID/collections/:collectionID/articles/:articleID", c.articleController.Show)
e.PUT("/api/workspaces/:workspaceID/collections/:collectionID/articles/:articleID", c.articleController.Update)
e.DELETE("/api/workspaces/:workspaceID/collections/:collectionID/articles/:articleID", c.articleController.Destroy)
}
```

---

## Authentication

Do not introduce custom authentication middleware. Instead, call `userService.Authenticate(extractToken(c))` at the top
of any controller handler that requires an authenticated user.

```go
func (a articleController) Index(c *echo.Context) error {
user, err := a.userService.Authenticate(extractToken(c))
if err != nil {
return err
}
// use `user` in downstream logic
}
```

---

## Input Validation

Perform input validation in the **service layer**, not in controllers. Use `dto.AppError` to return user-friendly validation errors.

### Email Validation

For email validation during signup/registration, use a compiled regex pattern at the package level:

```go
package service

import (
	"fmt"
	"regexp"

	"yourmodule/internal/dto"
)

var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)

func (u userService) Register(email string, password string) (string, error) {
	email = strings.TrimSpace(strings.ToLower(email))
	if !emailRegex.MatchString(email) {
		return "", dto.AppError(fmt.Errorf("invalid email format"))
	}
	// ... rest of registration logic
}
```

**Rules:**

- Compile regex patterns once at package level using `regexp.MustCompile` for performance.
- Always normalize email (trim whitespace, lowercase) before validation.
- Return `dto.AppError` for validation failures so they map to HTTP 400 responses.
- Keep validation logic in services, not controllers — controllers only bind and delegate.

---

## Multitenancy

When the data model must support multitenancy, use the **membership model** approach.

### `model/membership.go`

```go
package model

import "time"

type Membership struct {
	ID          string         `gorm:"primarykey" json:"id"`
	CreatedAt   time.Time      `json:"createdAt"`
	UpdatedAt   time.Time      `json:"updatedAt"`
	UserID      string         `json:"userID"`
	User        User           `json:"user"`
	WorkspaceID string         `json:"workspaceID"`
	Role        MembershipRole `json:"role"`
}

type MembershipRole string

const (
	MembershipRoleOwner  MembershipRole = "OWNER"
	MembershipRoleEditor MembershipRole = "EDITOR"
)
```

---

## Frontend Static Hosting

To serve a compiled frontend (e.g., a SPA) from the Go binary, embed the `app/` dist files and serve them via Echo's
static middleware.

### Changes to `cmd/main.go`

```go
import (
"embed"
"net/http"

"github.com/labstack/echo/v5/middleware"
)

//go:embed app
var appDistribution embed.FS

// Inside main(), after setting up routes:
e.Use(middleware.StaticWithConfig(middleware.StaticConfig{
Root:       "app",
Index:      "index.html",
HTML5:      true,
Filesystem: appDistribution,
}))
```

Also create the `cmd/app/` directory (this is where the built frontend files are placed).

### `build.sh`

This script pulls the latest frontend, builds it, copies the output into the backend, and redeploys via Docker Compose.

```bash
#!/bin/bash
cd ../frontend
git reset --hard HEAD
git pull
yarn
yarn build
cp -r dist/* ../backend/cmd/app
cd ../backend
git pull
docker compose up -d --build --no-cache
```

Make the script executable: `chmod +x build.sh`

---

## Kubernetes

When adding Kubernetes support, ask the developer for:

- `application-name` — the service/resource name
- `application-domain` — the public domain (e.g., `myapp.example.com`)
- `masterip` — the IP address of the master node

Then generate `k8s.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: application-name
  namespace: default
spec:
  ports:
    - name: http
      port: 3000
      targetPort: 3000
  externalName: masterip
---
apiVersion: v1
kind: Endpoints
metadata:
  name: application-name
  namespace: default
subsets:
  - addresses:
      - ip: masterip
    ports:
      - name: http
        port: 3000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: application-name-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: traefik
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - application-domain.com
      secretName: application-name-tls
  rules:
    - host: application-domain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: application-name
                port:
                  number: 3000
```

Keep in mind to use port from docker-compose in Kubernetes manifest!

---

## Swagger / OpenAPI Documentation

Expose interactive API documentation using Swagger UI. Store the OpenAPI spec and HTML page in `internal/controller/resources/`.

### File Structure

```
internal/controller/
  resources/
    openapi.yaml      # OpenAPI 3.0 specification
    swagger.html      # Swagger UI HTML page
  swagger.go          # Controller serving docs
scripts/
  generateSwagger.sh  # Script to regenerate OpenAPI spec
```

### Swagger UI HTML (`internal/controller/resources/swagger.html`)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>API Documentation</title>
    <link rel="stylesheet" type="text/css" href="https://unpkg.com/swagger-ui-dist@5/swagger-ui.css">
</head>
<body>
    <div id="swagger-ui"></div>
    <script src="https://unpkg.com/swagger-ui-dist@5/swagger-ui-bundle.js"></script>
    <script>
        window.onload = function() {
            SwaggerUIBundle({
                url: "/api/docs/openapi.yaml",
                dom_id: '#swagger-ui',
                presets: [
                    SwaggerUIBundle.presets.apis,
                    SwaggerUIBundle.SwaggerUIStandalonePreset
                ],
                layout: "BaseLayout"
            });
        };
    </script>
</body>
</html>
```

### Swagger Controller (`controller/swagger.go`)

```go
package controller

import (
	"embed"
	"net/http"

	"github.com/labstack/echo/v5"
)

//go:embed resources/openapi.yaml resources/swagger.html
var resources embed.FS

type SwaggerController interface {
	Spec(c *echo.Context) error
	UI(c *echo.Context) error
}

type swaggerController struct{}

func newSwaggerController() SwaggerController {
	return &swaggerController{}
}

func (s swaggerController) Spec(c *echo.Context) error {
	data, err := resources.ReadFile("resources/openapi.yaml")
	if err != nil {
		return err
	}
	return c.Blob(http.StatusOK, "application/yaml", data)
}

func (s swaggerController) UI(c *echo.Context) error {
	data, err := resources.ReadFile("resources/swagger.html")
	if err != nil {
		return err
	}
	return c.HTML(http.StatusOK, string(data))
}
```

### Route Registration

Register swagger routes in `controller/controllers.go`:

```go
e.GET("/api/docs", c.swaggerController.UI)
e.GET("/api/docs/openapi.yaml", c.swaggerController.Spec)
```

### Regenerating OpenAPI Spec (`scripts/generateSwagger.sh`)

Use this script to regenerate the OpenAPI specification by analyzing the codebase:

```bash
#!/bin/bash
junie --auth="$JUNIE_API_KEY" "Please analyze Go codebase and update internal/controller/resources/openapi.yaml"
```

Make executable: `chmod +x scripts/generateSwagger.sh`

---

## Git & Tooling

### `.gitignore`

```gitignore
# Binaries
/main
*.exe
*.out

# Environment
.env

# Go module cache
vendor/

# IDE
.idea/
.vscode/
*.swp
```

### Dependencies

| Library                                    | Purpose               |
|--------------------------------------------|-----------------------|
| `github.com/labstack/echo/v5`              | HTTP framework        |
| `gorm.io/gorm` + `gorm.io/driver/postgres` | ORM                   |
| `github.com/pressly/goose/v3`              | Database migrations   |
| `github.com/sirupsen/logrus`               | Structured logging    |
| `github.com/joho/godotenv`                 | `.env` file loading   |
| `github.com/golang-jwt/jwt/v4`             | JWT encoding/decoding |
| `golang.org/x/crypto/bcrypt`               | Password hashing      |

### Coding and work rules

- No abbreviations in variable names. Prefer full names: `transaction`, `workspaceMembership`, `article`.
- No inline comments unless documenting an interface or non-obvious behavior.
- Keep structs and interfaces in the same file as their constructor.
- For Echo v5 controller methods, use `*echo.Context` instead of `echo.Context` - pointers are required since v5 version
  in methods like `.GET` or `.POST` or others...
- Always use context7 for researching libraries documentation - Gorm, Labstack Echo, and others
- Don't run or compile application - developer will do that manually. Only write code.
- Use git semantic names commit messages. Example prefixes are `chore:`, `feat:`, `fix:`...
- Don't use any `git` commands - don't commit anything.