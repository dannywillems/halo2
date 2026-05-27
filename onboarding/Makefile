.PHONY: help
help: ## Ask for help!
	@grep -E '^[a-zA-Z0-9_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | \
		awk 'BEGIN {FS = ":.*?## "}; \
		{printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'

.PHONY: install
install: ## Install dependencies
	npm install

.PHONY: dev
dev: ## Run docs dev server
	npx docusaurus start

.PHONY: build
build: ## Build the static site
	npx docusaurus build

.PHONY: preview
preview: ## Serve the built site locally
	npx docusaurus serve

.PHONY: clean
clean: ## Clear Docusaurus build cache
	npx docusaurus clear

.PHONY: check-format
check-format: ## Check Markdown / TS formatting
	npx prettier --check .

.PHONY: format
format: ## Format Markdown / TS files
	npx prettier --write .
