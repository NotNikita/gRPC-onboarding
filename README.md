# Go Microservices - Pattern 2 + Buf Architecture Plan

## 1. HIGH-LEVEL ARCHITECTURE

### System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    GitLab Instance                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────┐      ┌──────────────────────┐    │
│  │  shared-protos repo  │      │ GoMicroservices repo │    │
│  │  (Contract Store)    │      │                      │    │
│  ├──────────────────────┤      ├──────────────────────┤    │
│  │ logger/              │      │ broker-service/      │    │
│  │ ├─ logger.proto      │      │ └─ go.mod (v1.2.3)   │    │
│  │ broker/              │◄─────┤    ↓ uses            │    │
│  │ ├─ broker.proto      │      │ logger.proto         │    │
│  │ generated/           │      │                      │    │
│  │ ├─ logger/           │      │ logger-service/      │    │
│  │ │  ├─ logger.pb.go   │      │ └─ go.mod (v1.2.3)   │    │
│  │ │  └─ logger_grpc.pb │      │    ↓ uses            │    │
│  │ ├─ broker/           │      │ broker.proto         │    │
│  │ │  ├─ broker.pb.go   │      │                      │    │
│  │ │  └─ broker_grpc.pb │      │ docker-compose.yml   │    │
│  │ buf.yaml             │      │ ├─ broker:50001      │    │
│  │ buf.lock             │      │ └─ logger:50001      │    │
│  │ .gitlab-ci.yml       │      │                      │    │
│  │ go.mod (generated)   │      └──────────────────────┘    │
│  │ v1.2.3 tag           │                                    │
│  │ v1.2.3-rc1 tag       │                                    │
│  └──────────────────────┘                                    │
│          ↑                                                   │
│         CI Pipeline:                                        │
│         • buf lint (proto validation)                       │
│         • buf breaking (breaking change detection)         │
│         • buf generate (code generation)                    │
│         • Tag v1.2.3-rc1 (on dev branch)                    │
│         • Tag v1.2.3 (on main branch)                       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Local Development                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Developer Machine:                                        │
│  ├─ shared-protos/ (cloned locally)                         │
│  │  ├─ logger.proto                                         │
│  │  ├─ broker.proto                                         │
│  │  └─ Makefile (buf lint, buf generate)                    │
│  │                                                          │
│  ├─ GoMicroservices/                                        │
│  │  ├─ broker-service/                                      │
│  │  │  └─ go.mod (replace => ../shared-protos)             │
│  │  └─ logger-service/                                      │
│  │     └─ go.mod (replace => ../shared-protos)             │
│  │                                                          │
│  docker-compose up                                         │
│  ├─ broker-service:8080 (HTTP) + 50001 (gRPC)             │
│  ├─ logger-service:7070 (HTTP) + 50001 (gRPC)             │
│  └─ postgres:5432 + mongo:27017 + rabbitmq:5672           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Key Components

| Component        | Role                                 | Ownership   | Technology           |
| ---------------- | ------------------------------------ | ----------- | -------------------- |
| `shared-protos`  | Single source of truth for contracts | Both teams  | Protobuf 3 + Buf CLI |
| `broker-service` | gRPC Server + HTTP Fiber API         | Broker Team | Go + gRPC + Fiber    |
| `logger-service` | gRPC Server + HTTP Fiber API         | Logger Team | Go + gRPC + Fiber    |
| `generated/`     | Auto-generated Go code               | CI/CD       | buf generate         |
| `.gitlab-ci.yml` | Validation & publishing              | Infra       | GitLab CI            |

---

## 2. ONBOARDING PLAN FOR 2 MICROSERVICES + GITLAB

### Phase 1: Setup Infrastructure (2 hours)

#### 1.1 Create shared-protos GitLab Repository

```bash
# On GitLab:
1. Create new project: "shared-protos"
2. Initialize with README
3. Clone locally

git clone https://gitlab.com/yourcompany/shared-protos.git
cd shared-protos
```

#### 1.2 Structure shared-protos Repo

```bash
# Create directory structure
mkdir -p logger broker generated/{logger,broker}

# Move proto files from existing services
cp ../GoMicroservices/logger-service/logs/logger.proto logger/
cp ../GoMicroservices/broker-service/logs/broker.proto broker/

# Create go.mod for generated code
cat > generated/go.mod << 'EOF'
module gitlab.com/yourcompany/shared-protos/generated

go 1.24

require (
    google.golang.org/grpc v1.78.0
    google.golang.org/protobuf v1.36.10
)
EOF
```

#### 1.3 Install Buf CLI

```bash
# macOS
brew install bufbuild/buf/buf

# Linux
curl -sSL https://github.com/bufbuild/buf/releases/download/v1.52.0/buf-Linux-x86_64 -o /usr/local/bin/buf
chmod +x /usr/local/bin/buf

# Verify
buf --version
```

#### 1.4 Create Buf Configuration

```yaml
# shared-protos/buf.yaml
version: v1

deps:
  - buf.build/googleapis/googleapis

lint:
  use:
    - DEFAULT
    - COMMENTS
  except:
    - ENUM_ZERO_VALUE_SUFFIX

breaking:
  use:
    - FILE
```

#### 1.5 Create Makefile

```makefile
# shared-protos/Makefile
.PHONY: lint breaking generate clean

lint:
	buf lint

breaking:
	buf breaking --against 'git://HEAD~1'

generate:
	buf generate

clean:
	rm -rf generated/*.pb.go generated/*_grpc.pb.go
```

#### 1.6 Test Locally

```bash
cd shared-protos
make lint          # Should pass
make generate      # Generate Go code
ls -la generated/logger/
ls -la generated/broker/
```

### Phase 2: GitLab CI/CD Pipeline Setup (1.5 hours)

#### 2.1 Create .gitlab-ci.yml

```yaml
# shared-protos/.gitlab-ci.yml
stages:
  - validate
  - generate
  - publish

variables:
  REGISTRY: registry.gitlab.com
  IMAGE_NAME: $REGISTRY/yourcompany/shared-protos

# Validate on every MR and commit
validate:
  stage: validate
  image: bufbuild/buf:latest
  script:
    - buf lint
    - buf breaking --against origin/main
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "dev"'
    - if: '$CI_COMMIT_BRANCH == "main"'

# Generate on dev and main branches
generate:
  stage: generate
  image: golang:1.24-alpine
  script:
    - apk add --no-cache git
    - buf generate
    - cd generated && go mod tidy
  artifacts:
    paths:
      - generated/
    expire_in: 1 day
  rules:
    - if: '$CI_COMMIT_BRANCH == "dev" || $CI_COMMIT_BRANCH == "main"'

# Compile check
compile_check:
  stage: generate
  image: golang:1.24-alpine
  dependencies:
    - generate
  script:
    - cd generated && go build ./...
  rules:
    - if: '$CI_COMMIT_BRANCH == "dev" || $CI_COMMIT_BRANCH == "main"'

# Tag RC version on dev branch
tag_rc:
  stage: publish
  image: alpine/git:latest
  script:
    - git config user.email "ci@gitlab.com"
    - git config user.name "CI Bot"
    - RC_TAG="v$(date +%Y.%m.%d)-rc-$(git rev-parse --short HEAD)"
    - git tag $RC_TAG
    - git push origin $RC_TAG
  only:
    - dev

# Tag stable version on main branch
tag_stable:
  stage: publish
  image: alpine/git:latest
  script:
    - git config user.email "ci@gitlab.com"
    - git config user.name "CI Bot"
    - STABLE_TAG="v$(date +%Y.%m.%d)"
    - git tag $STABLE_TAG
    - git push origin $STABLE_TAG
  only:
    - main
```

#### 2.2 Setup Branch Protection Rules

```
GitLab → Settings → Repository → Protected branches

Protected branch: main
├─ Require approval: 1
├─ Dismiss approvals: Yes
├─ Require all status checks to pass: Yes
│  ├─ validate (buf lint + buf breaking)
│  ├─ generate
│  └─ compile_check
├─ Allow force push: No

Protected branch: dev
├─ Require approval: 0 (Allow direct pushes)
├─ Allow force push: Yes (For iteration)
```

### Phase 3: Integrate Microservices (1 hour)

#### 3.1 Update broker-service/go.mod

```go
module broker

go 1.24

require (
    gitlab.com/yourcompany/shared-protos/generated v0.0.1
    github.com/gofiber/fiber/v2 v2.52.6
    google.golang.org/grpc v1.78.0
)

// Local development only - remove before merge!
replace gitlab.com/yourcompany/shared-protos/generated => ../../shared-protos/generated
```

#### 3.2 Update logger-service/go.mod

```go
module logger

go 1.24

require (
    gitlab.com/yourcompany/shared-protos/generated v0.0.1
    github.com/gofiber/fiber/v2 v2.52.6
    google.golang.org/grpc v1.78.0
)

// Local development only - remove before merge!
replace gitlab.com/yourcompany/shared-protos/generated => ../../shared-protos/generated
```

#### 3.3 Update Imports

```go
// broker-service/api/handlers.go
import "gitlab.com/yourcompany/shared-protos/generated/logger"

// logger-service/cmd/main.go
import "gitlab.com/yourcompany/shared-protos/generated/logger"
```

#### 3.4 Test Locally

```bash
cd GoMicroservices/broker-service
go mod tidy
go build ./...

cd ../logger-service
go mod tidy
go build ./...

docker-compose up
```

### Phase 4: Test CI/CD Pipeline (30 min)

#### 4.1 Push to shared-protos dev branch

```bash
cd shared-protos
git add .
git commit -m "Initial proto setup with broker and logger"
git push origin dev
```

#### 4.2 Check GitLab CI Pipeline

```
Go to: GitLab → shared-protos → CI/CD → Pipelines
├─ ✅ validate (buf lint passed)
├─ ✅ generate (Go code generated)
├─ ✅ compile_check (Generated code compiles)
└─ ✅ tag_rc (v2025.01.15-rc-abc123f tagged)
```

#### 4.3 Verify Tag Created

```bash
git fetch --tags
git tag -l | grep rc
# Output: v2025.01.15-rc-abc123f
```

---

## 3. DEVELOPER GUIDE: CONTRACT UPDATES & RELEASE CANDIDATES

### Scenario: Adding "UpdateLog" Endpoint to Logger Service

### Step 1: Clone & Setup Local Environment (5 min)

```bash
# Clone both repos
git clone https://gitlab.com/yourcompany/shared-protos.git
git clone https://gitlab.com/yourcompany/GoMicroservices.git

# Create workspace structure
mkdir -p ~/microservices-dev
cd ~/microservices-dev
ln -s ~/shared-protos shared-protos
ln -s ~/GoMicroservices GoMicroservices

# Verify buf is installed
buf --version
```

### Step 2: Create Feature Branch (2 min)

```bash
cd shared-protos

# Create feature branch from dev
git checkout dev
git pull origin dev
git checkout -b feature/logger-update-endpoint
```

### Step 3: Update Proto Contract (5 min)

```protobuf
# shared-protos/logger/logger.proto

syntax = "proto3";
package logger;

option go_package = "gitlab.com/yourcompany/shared-protos/generated/logger";

service LoggerService {
    rpc LogInfo (LogRequest) returns (LogResponse);
    rpc UpdateLog (UpdateLogRequest) returns (UpdateLogResponse);  // ← NEW
}

message LogRequest {
    string name = 1;
    string data = 2;
}

message LogResponse {
    string result = 1;
}

// ← NEW MESSAGES
message UpdateLogRequest {
    string id = 1;
    string data = 2;
}

message UpdateLogResponse {
    bool success = 1;
    string message = 2;
}
```

### Step 4: Validate & Generate Locally (3 min)

```bash
# Lint proto files
make lint
# Output: No linting issues.

# Check for breaking changes
make breaking
# Output: No breaking changes detected.

# Generate Go code
make generate
# Output: Generated files in generated/logger/

# Verify generated code
ls -la generated/logger/
# logger.pb.go
# logger_grpc.pb.go
```

### Step 5: Test Generated Code Compiles (2 min)

```bash
cd generated
go build ./...
# Success!

cd ../..
```

### Step 6: Push Feature Branch & Create MR (5 min)

```bash
git add logger/logger.proto
git commit -m "feat(logger): add UpdateLog endpoint"
git push origin feature/logger-update-endpoint
```

**On GitLab:**

1. Go to Merge Requests → New Merge Request
2. Base: `dev` ← Target: `feature/logger-update-endpoint`
3. Title: "feat(logger): add UpdateLog endpoint"
4. Description:

   ```
   ## Changes
   - Added UpdateLog RPC method to LoggerService
   - New messages: UpdateLogRequest, UpdateLogResponse

   ## Breaking Changes
   None

   ## Affected Services
   - logger-service (implementation needed)
   ```

5. Click "Create Merge Request"

### Step 7: Review & Merge (10 min)

**Pipeline runs automatically:**

```
✅ validate    (buf lint + buf breaking)
✅ generate    (buf generate)
✅ compile_check (Go code compiles)
```

**After review is approved:**

- Merge to `dev` branch
- Pipeline auto-generates RC tag: `v2025.01.15-rc-abc123f`

### Step 8: Update Logger Service Implementation (30 min)

```bash
# Now implement the new endpoint
cd GoMicroservices/logger-service

# Check that generated code is available
go mod tidy
go get gitlab.com/yourcompany/shared-protos/generated@v2025.01.15-rc-abc123f
```

**Implement the handler:**

```go
// logger-service/service/grpc.go

import (
    "context"
    "fmt"
    "logger/logs"
)

type GRPCLogServer struct {
    logs.UnimplementedLoggerServiceServer
    logService *LoggerService
}

// Existing method
func (l *GRPCLogServer) LogInfo(ctx context.Context, req *logs.LogRequest) (*logs.LogResponse, error) {
    // ...existing code...
}

// NEW method - implement here
func (l *GRPCLogServer) UpdateLog(ctx context.Context, req *logs.UpdateLogRequest) (*logs.UpdateLogResponse, error) {
    // Validation
    if req.Id == "" {
        return &logs.UpdateLogResponse{
            Success: false,
            Message: "ID is required",
        }, nil
    }

    // Update in MongoDB
    err := l.logService.UpdateLog(ctx, req.Id, req.Data)
    if err != nil {
        return &logs.UpdateLogResponse{
            Success: false,
            Message: fmt.Sprintf("Failed to update log: %v", err),
        }, nil
    }

    return &logs.UpdateLogResponse{
        Success: true,
        Message: "Log updated successfully",
    }, nil
}
```

**Implement the service method:**

```go
// logger-service/service/logs.go

import (
    "context"
    "go.mongodb.org/mongo-driver/v2/bson"
)

func (l *LoggerService) UpdateLog(ctx context.Context, id string, data string) error {
    // Parse ID
    objID, err := primitive.ObjectIDFromHex(id)
    if err != nil {
        return err
    }

    // Update in MongoDB
    filter := bson.M{"_id": objID}
    update := bson.M{
        "$set": bson.M{
            "data":       data,
            "updated_at": time.Now(),
        },
    }

    result, err := l.collection.UpdateOne(ctx, filter, update)
    if err != nil {
        return err
    }

    if result.MatchedCount == 0 {
        return fmt.Errorf("log not found")
    }

    return nil
}
```

### Step 9: Create Feature Branch for Implementation (5 min)

```bash
cd GoMicroservices
git checkout -b feature/logger-update-implementation
git add logger-service/
git commit -m "feat(logger-service): implement UpdateLog endpoint"
git push origin feature/logger-update-implementation
```

### Step 10: Create MR & Get Review (10 min)

**On GitLab → GoMicroservices:**

```
New MR:
Base: main ← feature/logger-update-implementation

Title: feat(logger-service): implement UpdateLog endpoint

Description:
## Implementation Details
- Implemented gRPC UpdateLog method
- Added MongoDB update operation
- Returns success/error response

## Testing
Tested with local gRPC client:
- UpdateLog with valid ID ✅
- UpdateLog with invalid ID ✅
- UpdateLog with missing data ✅

## Dependencies
Requires shared-protos >= v2025.01.15-rc-abc123f
```

````

### Step 11: Approve & Merge (5 min)

After 1 approval:
- ✅ Merge to `main`
- Notify logger service is ready

### Step 12: Promote RC to Stable Release (2 min)

Once implementation is tested & verified:

```bash
cd shared-protos
git checkout main
git pull origin main

# The RC tag already exists, just create stable version
git tag v2025.01.15
git push origin v2025.01.15
````

Or automated: When dev gets merged to main, CI auto-tags `v2025.01.15`

### Step 13: Broker Service Can Now Use Stable Version (5 min)

If broker-service needs UpdateLog endpoint:

```bash
cd GoMicroservices/broker-service

# Update to stable version
go get gitlab.com/yourcompany/shared-protos/generated@v2025.01.15

# Implement if needed
go mod tidy
```

---

## TIMELINE SUMMARY

| Phase                       | Time      | Owner            |
| --------------------------- | --------- | ---------------- |
| 1. Infrastructure setup     | 2 hrs     | DevOps/Tech Lead |
| 2. CI/CD pipeline setup     | 1.5 hrs   | DevOps/Tech Lead |
| 3. Microservice integration | 1 hr      | Both teams       |
| 4. Test CI/CD               | 30 min    | Both teams       |
| **TOTAL SETUP**             | **5 hrs** | -                |
| **Per feature update**      | **1 hr**  | Developer        |

---

## CHECKLISTS

### Pre-Launch Checklist

- [ ] shared-protos repo created on GitLab
- [ ] buf CLI installed locally on all devs
- [ ] buf.yaml configured
- [ ] .gitlab-ci.yml implemented
- [ ] Branch protection rules set (main + dev)
- [ ] broker-service go.mod updated with dependency
- [ ] logger-service go.mod updated with dependency
- [ ] Local test: make lint + generate successful
- [ ] Local test: docker-compose up runs all services
- [ ] CI pipeline tested with dev branch push
- [ ] RC tag auto-created by CI
- [ ] Stable tag auto-created when merged to main

### Per-Feature Checklist

- [ ] Feature branch created from dev
- [ ] Proto contract updated
- [ ] `make lint` passed
- [ ] `make breaking` passed (no breaking changes)
- [ ] `make generate` successful
- [ ] Generated code compiles
- [ ] MR created with description
- [ ] Code review approved
- [ ] Merged to dev
- [ ] RC tag created by CI
- [ ] Implementation started using RC tag
- [ ] Implementation tested locally
- [ ] Implementation MR created
- [ ] Implementation review approved
- [ ] Merged to main
- [ ] Stable tag created by CI (if not auto)
- [ ] Other services can now upgrade to stable

---

## COMMANDS QUICK REFERENCE

### Developer Commands

```bash
# Setup (first time)
cd shared-protos
make lint
make generate

# During development
make lint              # Validate proto syntax
make breaking          # Check for breaking changes
make generate          # Create/update Go code

# Verify
cd generated && go build ./...

# Git workflow
git checkout -b feature/my-feature
# ... edit proto ...
make generate
git add .
git commit -m "feat: description"
git push origin feature/my-feature
# Create MR on GitLab
```

### CI/CD Pipeline Commands (Auto-run)

```
On dev branch push:
• buf lint ✓
• buf breaking ✓
• buf generate ✓
• go build ✓
• auto-tag v2025.01.15-rc-xxx ✓

On main branch push (from MR merge):
• buf lint ✓
• buf breaking ✓
• buf generate ✓
• go build ✓
• auto-tag v2025.01.15 ✓
```

---

## TROUBLESHOOTING

### Issue: "buf: command not found"

```bash
# Solution: Install buf
brew install bufbuild/buf/buf
```

### Issue: "buf breaking" shows false breaking changes

```bash
# Solution: Ensure main branch is up to date
git fetch origin main
make breaking --against 'git://origin/main'
```

### Issue: Generated code not compiling

```bash
# Solution: Check proto syntax and buf.yaml
make lint
# Fix any linting errors
make generate
```

### Issue: RC tag not created

```bash
# Solution: Check .gitlab-ci.yml is committed
# Push to dev branch, check GitLab CI/CD pipeline
# Look for "tag_rc" job in pipeline
```

### Issue: Service won't use updated proto

```bash
# Solution: Update go.mod and go.sum
cd microservice
go get -u gitlab.com/yourcompany/shared-protos/generated@v2025.01.15-rc-xxx
go mod tidy
go build ./...
```
