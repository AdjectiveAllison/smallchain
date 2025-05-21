# Working with Gin Servers

This guide provides best practices for working with Gin HTTP servers in the Agent Control Plane codebase.

## Overview

Gin provides a powerful context object (`*gin.Context`) that can be used to read request information and send response data. However, it's important to structure Gin endpoints properly to ensure maintainability, testability, and to avoid common pitfalls.

## Key Principles

1. **Keep handlers thin** - HTTP handlers should focus only on HTTP concerns
2. **Separate business logic** - Move all business logic to separate functions
3. **Pass request context** - Always pass `c.Request.Context()` to business logic functions
4. **Return early after responses** - Always `return` immediately after calling response methods on `c`
5. **Centralize error handling** - Use consistent error handling patterns

## Common Gin Patterns

### Accessing Request Context

```go
ctx := c.Request.Context()
```

### Sending JSON Responses

```go
c.JSON(http.StatusOK, gin.H{"message": "Hello, World!"})
```

## Anti-Pattern Example

The following pattern should be avoided as it makes code hard to maintain and test:

```go
// ❌ AVOID: This approach mixes HTTP and business logic
func (s *Server) createUser(c *gin.Context) {
    var user User
    if err := c.ShouldBindJSON(&user); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    // Business logic mixed with HTTP concerns
    // ... many lines of code
    if someErrorCondition {
        c.JSON(http.StatusBadRequest, gin.H{"error": "some error"})
        return
    }
    // ... many more lines of code
    if someOtherErrorCondition {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "some other error"})
        return
    }
    // many more lines of code
    c.JSON(http.StatusOK, gin.H{"message": "User created successfully"})
}
```

This approach has several issues:
- Need to remember to `return` after calling methods on `c` that generate responses
- Hard to tell from the signature what the method's input/outputs are
- Duplicated error handling code makes it easy to introduce inconsistencies
- Side-effecting functions (methods on `c`) are woven throughout what could be mostly pure business logic

## Recommended Pattern

Instead, separate business logic from HTTP concerns:

```go
// ✅ RECOMMENDED: Pure business logic function with clean signature
func createUser(ctx context.Context, user User) (*User, error) {
    // ... business logic without HTTP concerns
    if someErrorCondition {
        return nil, errors.New("some error")
    }
    // ... more business logic
    if someOtherErrorCondition {
        return nil, errors.New("some other error")
    }
    // ... finalize business logic
    return &user, nil
}

// Thin HTTP handler that only deals with HTTP concerns
func (s *Server) createUserHandler(c *gin.Context) {
    var user User
    // Handle request parsing
    if err := c.ShouldBindJSON(&user); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    // Call business logic with context
    user, err := createUser(c.Request.Context(), user)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    
    // Send response
    c.JSON(http.StatusOK, user)
}
```

This approach provides several benefits:
- Decouples business logic from HTTP/Gin interfaces
- Enables business logic to be used in different contexts (e.g., CLI, queue workers)
- Methods return standard error types for consistent error handling
- Type signatures make it easy to understand inputs and outputs
- Business logic is easier to test without mocking Gin

## Implementation Example

ACP implements this pattern in the API server (`internal/server/server.go`). For example, notice how routes are registered and handlers are kept thin:

```go
// registerRoutes sets up all API endpoints
func (s *APIServer) registerRoutes() {
    // Health check endpoint (unversioned)
    s.router.GET("/status", s.getStatus)

    // API v1 routes
    v1 := s.router.Group("/v1")

    // Task endpoints
    tasks := v1.Group("/tasks")
    tasks.GET("", s.listTasks)
    tasks.GET("/:id", s.getTask)
    tasks.POST("", s.createTask)
    
    // Agent endpoints
    agents := v1.Group("/agents")
    agents.GET("", s.listAgents)
    agents.GET("/:name", s.getAgent)
    agents.POST("", s.createAgent)
    agents.PUT("/:name", s.updateAgent)
    agents.DELETE("/:name", s.deleteAgent)
}
```

## Best Practices

1. **Handler Organization**:
   - Register all routes in a central location (`registerRoutes`)
   - Group related endpoints (`v1.Group("/tasks")`)
   - Use consistent naming (`listTasks`, `getTask`, `createTask`)

2. **Error Handling**:
   - Use consistent HTTP status codes for different error types
   - Format error responses consistently (`c.JSON(status, gin.H{"error": err.Error()})`)
   - Consider creating error helper functions for common patterns

3. **Input Validation**:
   - Validate input in the handler before passing to business logic
   - Return early with a 400 status for validation errors
   - Consider using struct tags for validation (`binding:"required"`)

4. **Context Propagation**:
   - Always pass `c.Request.Context()` to business logic functions
   - Respect cancellation in long-running operations
   - Use context for request-scoped values like trace IDs

5. **Testing**:
   - Test business logic functions directly without Gin
   - Use httptest for testing handlers
   - Consider table-driven tests for handlers with different inputs

## Conclusion

By keeping Gin handlers thin and focused on HTTP concerns, you'll create more maintainable, testable, and reusable code. This pattern is particularly important in a Kubernetes operator where the same business logic might be used in different contexts (API server, controllers, CLI tools, etc.).