# Makefile for InstructLab GitOps
# Provides utility functions for managing the GitOps workflow

# Variables
KUBECONFORM_VERSION ?= 0.6.1
KUSTOMIZE_VERSION ?= 5.1.1
ARGOCD_VERSION ?= 2.8.4

# Default namespace for ArgoCD
ARGOCD_NAMESPACE ?= openshift-gitops
# Default namespace for pipeline
PIPELINE_NAMESPACE ?= data-science-project
# Default environment
ENV ?= dev

# Detect operating system
UNAME_S := $(shell uname -s)
ifeq ($(UNAME_S),Linux)
    OS_NAME := linux
    SED_INPLACE := sed -i
    INSTALL_CMD := sudo mv
endif
ifeq ($(UNAME_S),Darwin)
    OS_NAME := darwin
    SED_INPLACE := sed -i ''
    INSTALL_CMD := sudo mv
    # On macOS, check if Homebrew is installed
    BREW_INSTALLED := $(shell command -v brew 2> /dev/null)
endif

.PHONY: help install-tools validate-all validate-operators validate-pipeline apply-operators apply-pipeline run-pipeline clean update-repos

help: ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

install-tools: ## Install required tools for validation and deployment
ifeq ($(OS_NAME),darwin)
ifdef BREW_INSTALLED
	@echo "Using Homebrew to install tools on macOS..."
	@brew install kustomize argocd
	@brew tap instrumenta/instrumenta
	@brew install kubeconform
else
	@echo "Installing kubeconform $(KUBECONFORM_VERSION) for macOS..."
	@curl -sL "https://github.com/yannh/kubeconform/releases/download/v$(KUBECONFORM_VERSION)/kubeconform-darwin-arm64.tar.gz" | tar xz
	@$(INSTALL_CMD) kubeconform /usr/local/bin/kubeconform
	
	@echo "Installing kustomize $(KUSTOMIZE_VERSION) for macOS..."
	@curl -sL "https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv$(KUSTOMIZE_VERSION)/kustomize_v$(KUSTOMIZE_VERSION)_darwin_arm64.tar.gz" | tar xz
	@$(INSTALL_CMD) kustomize /usr/local/bin/kustomize
	
	@echo "Installing argocd $(ARGOCD_VERSION) for macOS..."
	@curl -sLO "https://github.com/argoproj/argo-cd/releases/download/v$(ARGOCD_VERSION)/argocd-darwin-arm64"
	@chmod +x argocd-darwin-arm64
	@$(INSTALL_CMD) argocd-darwin-arm64 /usr/local/bin/argocd
endif
else
	@echo "Installing kubeconform $(KUBECONFORM_VERSION) for Linux..."
	@curl -sL "https://github.com/yannh/kubeconform/releases/download/v$(KUBECONFORM_VERSION)/kubeconform-linux-amd64.tar.gz" | tar xz
	@$(INSTALL_CMD) kubeconform /usr/local/bin/kubeconform
	
	@echo "Installing kustomize $(KUSTOMIZE_VERSION) for Linux..."
	@curl -sL "https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv$(KUSTOMIZE_VERSION)/kustomize_v$(KUSTOMIZE_VERSION)_linux_amd64.tar.gz" | tar xz
	@$(INSTALL_CMD) kustomize /usr/local/bin/kustomize
	
	@echo "Installing argocd $(ARGOCD_VERSION) for Linux..."
	@curl -sLO "https://github.com/argoproj/argo-cd/releases/download/v$(ARGOCD_VERSION)/argocd-linux-amd64"
	@chmod +x argocd-linux-amd64
	@$(INSTALL_CMD) argocd-linux-amd64 /usr/local/bin/argocd
endif

validate-all: validate-operators validate-pipeline ## Validate all Kubernetes manifests

validate-operators: ## Validate operator manifests
	@echo "Validating operator manifests..."
	@kustomize build operators/overlays/$(ENV) | kubeconform -strict -ignore-missing-schemas -schema-location default

validate-pipeline: ## Validate pipeline manifests
	@echo "Validating pipeline manifests..."
	@kustomize build overlays/$(ENV) | kubeconform -strict -ignore-missing-schemas -schema-location default

apply-operators: validate-operators ## Apply operator manifests to the cluster
	@echo "Applying operator manifests to the cluster..."
	@oc apply -f argocd/app-of-apps.yaml -n $(ARGOCD_NAMESPACE)
	@echo "Operators application deployed. Check status with: argocd app get openshift-ai-stack -n $(ARGOCD_NAMESPACE)"

apply-pipeline: validate-pipeline ## Apply pipeline manifests to the cluster
	@echo "Applying pipeline manifests to the cluster..."
	@oc apply -f argocd/instructlab-pipeline-app.yaml -n $(ARGOCD_NAMESPACE)
	@echo "Pipeline application deployed. Check status with: argocd app get instructlab-pipeline -n $(ARGOCD_NAMESPACE)"

run-pipeline: ## Run the pipeline with parameters from scripts/pipeline-parameters.env
	@echo "Running InstructLab pipeline..."
	@scripts/create-pipeline-run.sh $(ENV)

clean: ## Clean up temporary files
	@echo "Cleaning up temporary files..."
	@find . -name "*.tmp" -delete
	@find . -name "*.bak" -delete

update-repos: ## Update Git repository URLs in all manifest files
	@echo "Updating Git repository URLs..."
ifeq ($(OS_NAME),darwin)
	@read -p "Enter your Git repository URL: " REPO_URL; \
	find argocd -name "*.yaml" -type f -exec $(SED_INPLACE) "s|repoURL: https://github.com/open-demos/nlit2025.git|repoURL: $$REPO_URL|g" {} \;
else
	@read -p "Enter your Git repository URL: " REPO_URL; \
	find argocd -name "*.yaml" -type f -exec $(SED_INPLACE) "s|repoURL: https://github.com/open-demos/nlit2025.git|repoURL: $$REPO_URL|g" {} \;
endif
	@echo "Repository URLs updated."

sync-all: ## Sync all ArgoCD applications
	@echo "Syncing all ArgoCD applications..."
	@argocd app sync openshift-ai-stack
	@argocd app sync instructlab-pipeline
