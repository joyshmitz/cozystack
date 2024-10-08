NAMESPACE=cozy-e2e-tests
NAME := sandbox
CLEAN := 1

include ../../../scripts/common-envs.mk


help: ## Show this help.
	@awk 'BEGIN {FS = ":.*?## "} /^[a-zA-Z_-]+:.*?## / {sub("\\\\n",sprintf("\n%22c"," "), $$2);printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}' $(MAKEFILE_LIST)
show:
	helm template -n $(NAMESPACE) $(NAME) .

apply: ## Create sandbox in existing Kubernetes cluster.
	helm template -n $(NAMESPACE) $(NAME) . | kubectl apply -f -

diff:
	helm template -n $(NAMESPACE) $(NAME) . | kubectl diff -f -

image: image-e2e-sandbox

image-e2e-sandbox:
	docker buildx build -f images/e2e-sandbox/Dockerfile ../../.. \
		--provenance false \
		--tag $(REGISTRY)/e2e-sandbox:$(call settag,$(TAG)) \
		--cache-from type=registry,ref=$(REGISTRY)/e2e-sandbox:latest \
		--platform linux/amd64,linux/arm64 \
		--cache-to type=inline \
		--metadata-file images/e2e-sandbox.json \
		--push=$(PUSH) \
		--load=$(LOAD)
	IMAGE="$(REGISTRY)/e2e-sandbox:$(call settag,$(TAG))@$$(yq e '."containerimage.digest"' images/e2e-sandbox.json -o json -r)" \
		yq -i '.e2e.image = strenv(IMAGE)' values.yaml
	rm -f images/e2e-sandbox.json

test: wait-for-sandbox ## Run the end-to-end tests in existing sandbox.
	cat ../../../hack/e2e.sh | kubectl exec -i -n $(NAMESPACE) deploy/cozystack-e2e-$(NAME) -- sh -c 'cat > /e2e.sh && chmod +x /e2e.sh'
	helm template -n cozy-system installer ../installer | kubectl exec -i -n $(NAMESPACE) deploy/cozystack-e2e-$(NAME) -- sh -c 'cat > /cozystack-installer.yaml'
	kubectl exec -ti -n $(NAMESPACE) deploy/cozystack-e2e-$(NAME) -- sh -c 'export COZYSTACK_INSTALLER_YAML=$$(cat /cozystack-installer.yaml) && /e2e.sh'

delete: ## Remove sandbox from existing Kubernetes cluster.
	kubectl delete deploy -n $(NAMESPACE) cozystack-e2e-$(NAME)

exec:  ## Opens an interactive shell in the sandbox container.
	kubectl exec -ti -n $(NAMESPACE) deploy/cozystack-e2e-$(NAME) -- bash

proxy: sync-hosts ## Enable a SOCKS5 proxy server; mirrord and gost must be installed.
	mirrord exec --target deploy/cozystack-e2e-sandbox --target-namespace cozy-e2e-tests -- gost -L=127.0.0.1:10080

login: ## Downloads the kubeconfig into a temporary directory and runs a shell with the sandbox environment; mirrord must be installed.
	mirrord exec --target deploy/cozystack-e2e-sandbox --target-namespace cozy-e2e-tests -- "$$SHELL"

sync-hosts:
	kubectl exec -n $(NAMESPACE) deploy/cozystack-e2e-$(NAME) -- sh -c 'kubectl get ing -A -o go-template='\''{{ "127.0.0.1 localhost\n"}}{{ range .items }}{{ range .status.loadBalancer.ingress }}{{ .ip }}{{ end }} {{ range .spec.rules }}{{ .host }}{{ end }}{{ "\n" }}{{ end }}'\'' > /etc/hosts'

wait-for-sandbox:
	kubectl wait deploy --for=condition=Progressing -n $(NAMESPACE) cozystack-e2e-$(NAME)
	kubectl wait pod --for=condition=Ready -n $(NAMESPACE) -l app=cozystack-e2e-$(NAME)
