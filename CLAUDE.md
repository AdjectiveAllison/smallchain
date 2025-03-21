# SmallChain Development Guide

## Go (Kubechain) Commands
- Build: `cd kubechain && make build`
- Format: `cd kubechain && make fmt`
- Lint: `cd kubechain && make lint`
- Run tests: `cd kubechain && make test`
- Run single test: `cd kubechain && go test -v ./internal/controller/llm -run TestLLMController`
- Run e2e tests: `cd kubechain && make test-e2e`

## TypeScript Commands
- Build: `cd ts && npm run build`
- Dev mode: `cd ts && npm run dev`
- Run tests: `cd ts && npm test`
- Run single test: `cd ts && npm test -- -t "ChainService constructor"`

## Code Style Guidelines
### Go
- Follow standard Go code style with `gofmt`
- Use meaningful error handling with context
- Use dependency injection for controllers
- Test with Ginkgo/Gomega framework
- Document public functions with godoc

### TypeScript
- Use 2-space indentation
- No semicolons (per prettier config)
- Double quotes for strings
- Strong typing with TypeScript interfaces
- Use ES6+ features (arrow functions, destructuring)
- Jest for testing

### General
- Descriptive variable/function names (camelCase in TS, CamelCase for exported Go)
- Use consistent error handling patterns within each language
- Add tests for new functionality
- Keep functions small and focused