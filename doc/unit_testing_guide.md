# TimeCapsule 项目单元测试指南

## 📋 目录
1. [测试框架和工具](#测试框架和工具)
2. [测试最佳实践](#测试最佳实践)
3. [项目测试架构](#项目测试架构)
4. [测试覆盖率](#测试覆盖率)
5. [Mock 和 Stub](#mock-和-stub)
6. [集成测试](#集成测试)
7. [CI/CD 集成](#cicd-集成)

---

## 🛠️ 测试框架和工具

### 1. 核心测试库

#### Go 标准库 (testing)
Go 自带的测试框架，功能强大且简单易用。

```go
import "testing"

func TestFunctionName(t *testing.T) {
    result := FunctionUnderTest()
    expected := "expected value"
    
    if result != expected {
        t.Errorf("expected %s, got %s", expected, result)
    }
}
```

#### testify (已包含在项目中)
项目已经依赖了 `github.com/stretchr/testify`，提供断言和 Mock 功能。

```go
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
)

func TestWithAssert(t *testing.T) {
    assert.Equal(t, "expected", "expected")
    assert.NotNil(t, someObject)
    assert.True(t, condition)
}
```

### 2. 推荐添加的测试工具

#### gomock - Mock 生成工具
```bash
go install github.com/golang/mock/mockgen@latest
```

#### go-sqlmock - 数据库 Mock
```bash
go get github.com/DATA-DOG/go-sqlmock
```

#### httptest - HTTP 测试
Go 标准库，用于测试 HTTP 服务器。

#### testcontainers - 集成测试
```bash
go get github.com/testcontainers/testcontainers-go
```

---

## 📚 测试最佳实践

### 1. 测试命名规范

```go
// 好的命名
func TestInsertRecord_Success(t *testing.T) {}
func TestInsertRecord_DuplicateKey(t *testing.T) {}
func TestInsertRecord_InvalidInput(t *testing.T) {}

// 不好的命名
func TestInsertRecord1(t *testing.T) {}
func TestInsertRecordTest(t *testing.T) {}
```

### 2. 测试结构 (AAA 模式)

```go
func TestInsertRecord(t *testing.T) {
    // Arrange (准备)
    testDB := setupTestDB(t)
    defer cleanupTestDB(t, testDB)
    
    file := &File{
        Name: "test.jpg",
        Path: "/test",
        Type: FileType_Image,
    }
    
    // Act (执行)
    err := InsertRecord(file)
    
    // Assert (断言)
    assert.NoError(t, err)
    assert.NotZero(t, file.Id)
}
```

### 3. 表驱动测试

```go
func TestValidateFileName(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    bool
        wantErr bool
    }{
        {
            name:    "valid filename",
            input:   "test.jpg",
            want:    true,
            wantErr: false,
        },
        {
            name:    "empty filename",
            input:   "",
            want:    false,
            wantErr: true,
        },
        {
            name:    "filename with path",
            input:   "/path/to/file.jpg",
            want:    false,
            wantErr: true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ValidateFileName(tt.input)
            if (err != nil) != tt.wantErr {
                t.Errorf("ValidateFileName() error = %v, wantErr %v", err, tt.wantErr)
                return
            }
            if got != tt.want {
                t.Errorf("ValidateFileName() = %v, want %v", got, tt.want)
            }
        })
    }
}
```

### 4. 测试辅助函数

```go
// setupTestDB 创建测试数据库
func setupTestDB(t *testing.T) *gorm.DB {
    t.Helper()
    
    dbPath := filepath.Join(os.TempDir(), fmt.Sprintf("test_db_%d", time.Now().UnixNano()))
    db, err := gorm.Open(sqlite.Open(dbPath), &gorm.Config{})
    require.NoError(t, err)
    
    err = db.AutoMigrate(&File{}, &FileDir{}, &Tags{})
    require.NoError(t, err)
    
    t.Cleanup(func() {
        os.RemoveAll(dbPath)
    })
    
    return db
}

// createTestFile 创建测试文件
func createTestFile(t *testing.T, name, content string) string {
    t.Helper()
    
    tmpFile := filepath.Join(os.TempDir(), name)
    err := os.WriteFile(tmpFile, []byte(content), 0644)
    require.NoError(t, err)
    
    t.Cleanup(func() {
        os.Remove(tmpFile)
    })
    
    return tmpFile
}
```

### 5. 并发测试

```go
func TestConcurrentAccess(t *testing.T) {
    var wg sync.WaitGroup
    errors := make(chan error, 100)
    
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            if err := SomeFunction(); err != nil {
                errors <- err
            }
        }(i)
    }
    
    wg.Wait()
    close(errors)
    
    for err := range errors {
        t.Errorf("concurrent error: %v", err)
    }
}
```

---

## 🏗️ 项目测试架构

### 1. 目录结构

```
time_capsule/
├── src/
│   ├── api/
│   │   ├── api.go
│   │   ├── apiDo.go
│   │   └── api_test.go          # API 测试
│   ├── cfg/
│   │   ├── config.go
│   │   └── config_test.go       # 配置测试
│   ├── db/
│   │   ├── dbModule.go
│   │   ├── dbOperate.go
│   │   └── db_test.go          # 数据库测试
│   ├── fileProcess/
│   │   ├── fileProcess.go
│   │   └── fileProcess_test.go  # 文件处理测试
│   └── util/
│       ├── util.go
│       └── util_test.go        # 工具函数测试
├── tests/
│   ├── integration/              # 集成测试
│   │   ├── api_test.go
│   │   └── workflow_test.go
│   └── testdata/               # 测试数据
│       ├── images/
│       ├── videos/
│       └── configs/
└── scripts/
    ├── test.sh                  # 测试脚本
    └── coverage.sh             # 覆盖率脚本
```

### 2. 测试分层

#### 单元测试 (Unit Tests)
- 测试单个函数或方法
- 不依赖外部系统
- 快速执行

```go
// src/util/util_test.go
func TestSysStatusSetWorking(t *testing.T) {
    SysStatusSetWorking("test operation")
    
    status := SysStatusGet()
    assert.True(t, status.Busy)
    assert.Contains(t, status.Status, "working")
}
```

#### 集成测试 (Integration Tests)
- 测试多个组件的交互
- 使用真实的数据库或文件系统
- 执行时间较长

```go
// tests/integration/api_test.go
func TestFileWorkflow(t *testing.T) {
    db := setupTestDB(t)
    api := setupTestAPI(t, db)
    
    file := createTestFile(t, "test.jpg", "test content")
    
    resp := api.ScanFile(file)
    assert.Equal(t, Resp_Success, resp.Code)
    
    files := api.GetFiles()
    assert.Len(t, files.Records, 1)
}
```

#### 端到端测试 (E2E Tests)
- 测试完整的用户流程
- 模拟真实用户操作

```go
// tests/e2e/user_workflow_test.go
func TestUserUploadAndTagFile(t *testing.T) {
    app := setupTestApp(t)
    defer app.Cleanup()
    
    user := app.CreateUser("testuser")
    file := app.UploadFile(user, "test.jpg")
    
    tag := app.CreateTag("vacation")
    app.AddTagToFile(user, file.ID, tag.ID)
    
    files := app.SearchFiles(user, "tag:vacation")
    assert.Len(t, files, 1)
}
```

---

## 📊 测试覆盖率

### 1. 生成覆盖率报告

```bash
# 生成覆盖率报告
go test -coverprofile=coverage.out ./...

# 查看覆盖率
go tool cover -func=coverage.out

# 生成 HTML 报告
go tool cover -html=coverage.out -o coverage.html
```

### 2. 覆盖率目标

| 组件类型 | 目标覆盖率 | 说明 |
|---------|-----------|------|
| 核心业务逻辑 | 80%+ | 文件处理、数据库操作 |
| API 层 | 70%+ | 请求处理、响应 |
| 工具函数 | 90%+ | 纯函数、无副作用 |
| 配置管理 | 60%+ | 配置加载、验证 |

### 3. 覆盖率脚本

```bash
#!/bin/bash
# scripts/coverage.sh

set -e

echo "Running tests with coverage..."
go test -coverprofile=coverage.out -covermode=atomic ./...

echo "Generating coverage report..."
go tool cover -func=coverage.out | tee coverage.txt

echo "Generating HTML report..."
go tool cover -html=coverage.out -o coverage.html

echo "Coverage report generated: coverage.html"

# 检查覆盖率阈值
COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | sed 's/%//')
echo "Total coverage: $COVERAGE%"

if (( $(echo "$COVERAGE < 70" | bc -l) )); then
    echo "Coverage $COVERAGE% is below 70% threshold"
    exit 1
fi
```

---

## 🎭 Mock 和 Stub

### 1. 使用 gomock

```go
//go:generate mockgen -source=db.go -destination=db_mock.go

type DB interface {
    Insert(record interface{}) error
    Find(record interface{}) error
}

type FileService struct {
    db DB
}

func (s *FileService) SaveFile(file *File) error {
    return s.db.Insert(file)
}

// 测试
func TestFileService_SaveFile(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()
    
    mockDB := NewMockDB(ctrl)
    mockDB.EXPECT().Insert(gomock.Any()).Return(nil)
    
    service := &FileService{db: mockDB}
    err := service.SaveFile(&File{Name: "test.jpg"})
    
    assert.NoError(t, err)
}
```

### 2. 使用 go-sqlmock 测试数据库

```go
func TestInsertRecord_WithMockDB(t *testing.T) {
    db, mock, err := sqlmock.New()
    require.NoError(t, err)
    defer db.Close()
    
    gormDB, err := gorm.Open(sqlite.Dialector{Conn: db}, &gorm.Config{})
    require.NoError(t, err)
    
    mock.ExpectBegin()
    mock.ExpectExec("INSERT INTO files").
        WithArgs("test.jpg", "/test", FileType_Image).
        WillReturnResult(sqlmock.NewResult(1, 1))
    mock.ExpectCommit()
    
    file := &File{
        Name: "test.jpg",
        Path: "/test",
        Type: FileType_Image,
    }
    
    err = gormDB.Create(file).Error
    assert.NoError(t, err)
    assert.Equal(t, uint(1), file.Id)
}
```

### 3. 使用 httptest 测试 API

```go
func TestAPI_GetFiles(t *testing.T) {
    router := setupTestRouter()
    
    req := httptest.NewRequest("GET", "/api/v1/files", nil)
    w := httptest.NewRecorder()
    
    router.ServeHTTP(w, req)
    
    assert.Equal(t, 200, w.Code)
    
    var resp RespData[[]File]
    err := json.Unmarshal(w.Body.Bytes(), &resp)
    assert.NoError(t, err)
    assert.Equal(t, Resp_Success, resp.Code)
}
```

---

## 🔗 集成测试

### 1. 数据库集成测试

```go
// tests/integration/db_test.go
func TestDatabaseIntegration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }
    
    dbPath := filepath.Join(os.TempDir(), "integration_test.db")
    defer os.RemoveAll(dbPath)
    
    err := Init(dbPath)
    require.NoError(t, err)
    
    file := &File{
        Name: "integration_test.jpg",
        Path: "/test",
        Type: FileType_Image,
    }
    
    err = InsertRecord(file)
    assert.NoError(t, err)
    
    result, err := GetRecordsById[File](file.Id)
    assert.NoError(t, err)
    assert.Len(t, result.Records, 1)
}
```

### 2. 文件系统集成测试

```go
func TestFileProcessingIntegration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }
    
    testDir := filepath.Join(os.TempDir(), "test_files")
    defer os.RemoveAll(testDir)
    
    os.MkdirAll(testDir, 0755)
    
    testFile := filepath.Join(testDir, "test.jpg")
    err := os.WriteFile(testFile, []byte("fake image data"), 0644)
    require.NoError(t, err)
    
    result, err := ScanFile(testFile)
    assert.NoError(t, err)
    assert.Equal(t, "test.jpg", result.Name)
}
```

### 3. API 集成测试

```go
func TestAPIWorkflow(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }
    
    app := setupTestApp(t)
    defer app.Stop()
    
    client := &http.Client{}
    baseURL := app.Server.URL
    
    resp, err := client.Get(baseURL + "/api/v1/files")
    assert.NoError(t, err)
    assert.Equal(t, 200, resp.StatusCode)
}
```

---

## 🚀 CI/CD 集成

### 1. GitHub Actions 配置

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Go
      uses: actions/setup-go@v4
      with:
        go-version: '1.23'
    
    - name: Download dependencies
      run: go mod download
    
    - name: Run tests
      run: go test -v -race -coverprofile=coverage.out ./...
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.out
        flags: unittests
        name: codecov-umbrella
    
    - name: Check coverage threshold
      run: |
        COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | sed 's/%//')
        echo "Coverage: $COVERAGE%"
        if (( $(echo "$COVERAGE < 70" | bc -l) )); then
          echo "Coverage $COVERAGE% is below 70% threshold"
          exit 1
        fi
```

### 2. Pre-commit Hook

```bash
#!/bin/bash
# .git/hooks/pre-commit

echo "Running tests..."
go test ./...

if [ $? -ne 0 ]; then
    echo "Tests failed. Commit aborted."
    exit 1
fi

echo "Running go vet..."
go vet ./...

if [ $? -ne 0 ]; then
    echo "Go vet found issues. Commit aborted."
    exit 1
fi

echo "Running gofmt..."
unformatted=$(gofmt -l .)
if [ -n "$unformatted" ]; then
    echo "Unformatted files found:"
    echo "$unformatted"
    echo "Please run 'gofmt -w .' to format."
    exit 1
fi

echo "All checks passed. Proceeding with commit."
```

---

## 📝 测试清单

### 单元测试清单
- [ ] 所有公共函数都有测试
- [ ] 错误路径都被测试
- [ ] 边界条件都被测试
- [ ] 并发安全性被测试
- [ ] 测试使用表驱动模式
- [ ] 测试命名清晰明确

### 集成测试清单
- [ ] 数据库操作被测试
- [ ] 文件系统操作被测试
- [ ] API 端点被测试
- [ ] 完整工作流被测试
- [ ] 测试数据被清理

### 测试质量清单
- [ ] 测试独立运行
- [ ] 测试快速执行
- [ ] 测试可重复执行
- [ ] 测试覆盖率达标
- [ ] 测试文档完整

---

## 🎯 推荐的测试策略

### 阶段 1: 核心功能测试 (第 1-2 周)
1. 为 `util` 包添加完整测试
2. 为 `cfg` 包添加配置测试
3. 为 `db` 包添加数据库操作测试

### 阶段 2: 业务逻辑测试 (第 3-4 周)
1. 为 `fileProcess` 包添加文件处理测试
2. 为 `api` 包添加 API 处理测试
3. 添加并发安全测试

### 阶段 3: 集成测试 (第 5-6 周)
1. 添加数据库集成测试
2. 添加文件系统集成测试
3. 添加 API 集成测试

### 阶段 4: CI/CD 集成 (第 7 周)
1. 配置 GitHub Actions
2. 设置覆盖率检查
3. 配置 pre-commit hooks

---

## 📚 参考资源

### 官方文档
- [Go Testing](https://golang.org/pkg/testing/)
- [testify](https://github.com/stretchr/testify)
- [gomock](https://github.com/golang/mock)

### 最佳实践
- [Effective Go: Testing](https://golang.org/doc/effective_go#testing)
- [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments#testing)

### 工具
- [go-sqlmock](https://github.com/DATA-DOG/go-sqlmock)
- [testcontainers-go](https://github.com/testcontainers/testcontainers-go)
- [gocov](https://github.com/axw/gocov)

---

## 💡 总结

建立完整的测试体系需要：

1. **分层测试**：单元测试、集成测试、端到端测试
2. **测试工具**：使用 testify、gomock、go-sqlmock 等工具
3. **覆盖率监控**：设置覆盖率目标和 CI 检查
4. **持续集成**：集成到 CI/CD 流程
5. **测试文化**：鼓励编写测试，将测试作为开发的一部分

通过这套测试方案，可以显著提升代码质量和项目稳定性。
