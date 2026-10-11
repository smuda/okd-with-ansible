REGISTRY_HOST=registry.okd4.example.com
DOCKER_HOST=tcp://192.168.40.20:2376
DOCKER_TLS_VERIFY=1
VOLUME_BASE_PATH=/share/kube-volumes/proxy
PULL_SECRET=../../pull-secret

PHONY: ca
ca:
	# Create the CA
	echo "Create CA certificate"
	mkdir -p ../../openshift-ca
	openssl genrsa \
		-out ../../openshift-ca/ca.key \
		4096
	openssl req \
		-x509 \
		-new \
		-nodes \
		-sha256 \
		-days 1825 \
		-key ../../openshift-ca/ca.key \
		-out ../../openshift-ca/ca.pem \
		-subj "/C=SE/O=Ayoy AB/CN=OKD Lab CA v42"

	# Create the registry certificate
	echo "Create registry certificate"
	openssl genrsa \
		-out ../../openshift-ca/$(REGISTRY_HOST).key \
		2048
	openssl req \
		-new \
		-key ../../openshift-ca/$(REGISTRY_HOST).key \
		-out ../../openshift-ca/$(REGISTRY_HOST).csr \
		-subj "/CN=$(REGISTRY_HOST)/O=Ayoy AB"
	echo "authorityKeyIdentifier=keyid,issuer\n\
basicConstraints=CA:FALSE\n\
keyUsage = digitalSignature, keyEncipherment\n\
subjectAltName = @alt_names\n\
\n\
[alt_names]\n\
DNS.1 = $(REGISTRY_HOST)\n" > ../../openshift-ca/$(REGISTRY_HOST).ext
	openssl x509 \
		-req \
		-in ../../openshift-ca/$(REGISTRY_HOST).csr \
		-out ../../openshift-ca/$(REGISTRY_HOST).pem \
		-CA ../../openshift-ca/ca.pem \
		-CAkey ../../openshift-ca/ca.key \
		-CAcreateserial \
		-days 825 \
		-sha256 \
		-extfile ../../openshift-ca/$(REGISTRY_HOST).ext
	rm ../../openshift-ca/$(REGISTRY_HOST).csr
	rm ../../openshift-ca/$(REGISTRY_HOST).ext

PHONY: pull
pull:
	DOCKER_HOST=$(DOCKER_HOST) \
    DOCKER_TLS_VERIFY=1 \
	docker pull registry:2

PHONY: clean
clean:
	DOCKER_HOST=$(DOCKER_HOST) \
    DOCKER_TLS_VERIFY=1 \
    docker stop proxy-quay.io || echo "Failed to stop"

	DOCKER_HOST=$(DOCKER_HOST) \
    DOCKER_TLS_VERIFY=1 \
    docker rm proxy-quay.io || echo "Failed to stop"

	DOCKER_HOST=$(DOCKER_HOST) \
    DOCKER_TLS_VERIFY=1 \
    docker rmi registry:2

PHONY: quay.io
quay.io: pull
	# Credentials are kept in shell variables so make does not echo them
	auth=$$(jq -r '.auths["quay.io"].auth // empty' $(PULL_SECRET) | base64 -d); \
	test -n "$$auth" || { echo "No quay.io credentials in $(PULL_SECRET)"; exit 1; }; \
	export REGISTRY_PROXY_USERNAME="$${auth%%:*}" REGISTRY_PROXY_PASSWORD="$${auth#*:}"; \
	DOCKER_HOST=$(DOCKER_HOST) \
    DOCKER_TLS_VERIFY=1 \
    docker run \
		-d \
		-p 5011:443 \
		-p 55011:5000 \
		--restart=unless-stopped \
		--name=proxy-quay.io \
		-e REGISTRY_PROXY_REMOTEURL="https://quay.io" \
		-e REGISTRY_PROXY_USERNAME \
		-e REGISTRY_PROXY_PASSWORD \
		-e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
		-v "$(VOLUME_BASE_PATH)/quay.io/certs":/certs \
		-v "$(VOLUME_BASE_PATH)/quay.io/registry":/var/lib/registry \
		-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.pem \
		-e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
		"registry:2" \

PHONY: docker.io
docker.io: pull
ifndef DOCKER_HUB_USERNAME
	$(error Environment variable DOCKER_HUB_USERNAME is not set)
endif
ifndef DOCKER_HUB_ACCESS_TOKEN
	$(error Environment variable DOCKER_HUB_ACCESS_TOKEN is not set)
endif
	DOCKER_HOST=$(DOCKER_HOST) \
    DOCKER_TLS_VERIFY=1 \
    docker run \
		-d \
		-p 5010:443 \
		-p 55010:5000 \
		--restart=unless-stopped \
		--name=proxy-docker.io \
		-e REGISTRY_PROXY_REMOTEURL="https://registry-1.docker.io" \
		-e REGISTRY_PROXY_USERNAME=$(DOCKER_HUB_USERNAME) \
		-e REGISTRY_PROXY_PASSWORD=$(DOCKER_HUB_ACCESS_TOKEN) \
		-e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
		-v "$(VOLUME_BASE_PATH)/docker.io/certs":/certs \
		-v "$(VOLUME_BASE_PATH)/docker.io/registry":/var/lib/registry \
		-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.pem \
		-e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
		"registry:2" \
