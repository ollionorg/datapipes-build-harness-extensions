SONAR_GO_TEST_ARGS ?= ./...

SONAR_ORG ?= cldcvr


.PHONY: sonar/go
## Run SonarCloud analysis for Go on Travis CI. This will not be run during local development.
sonar/go: go/gosec-install
	@echo "-> Starting sonar/go"
	@echo "--> Starting go test"
	go test -coverprofile=coverage.out -json ${SONAR_GO_TEST_ARGS} | tee report.json | grep -v '"Action":"output"'
	@echo "--> Running gosec"
	gosec -fmt sonarqube -out gosec.json -no-fail ./...
	@echo "---> gosec gosec.json"
	@cat gosec.json
	@echo "--> Running sonar-scanner"
	unset SONARQUBE_SCANNER_PARAMS
	sonar-scanner --debug


.PHONY: sonar/go/prow
## Run SonarCloud analysis for Go on Prow CI. This will not be run during local development.
sonar/go/prow: SHELL:=/bin/bash
sonar/go/prow: export SONAR_TOKEN="$(shell cat /etc/sonarcloud/token 2> /dev/null)"
sonar/go/prow:
	@echo "Verify sonar token from kube secret has been mounted" | tee -a "${ARTIFACT_DIR}/sonar.log"
	@if [[ -z "${SONAR_TOKEN}" ]] ; then echo "---> ERROR: SONAR_TOKEN is not set" | tee -a "${ARTIFACT_DIR}/sonar.log" ; exit 1 ; fi

	@echo "Running go test" | tee -a "${ARTIFACT_DIR}/sonar.log"
	go test -coverprofile=coverage.out -json ${SONAR_GO_TEST_ARGS} | tee -a report.json "${ARTIFACT_DIR}/sonar_gotest.json"
	@cp coverage.out "${ARTIFACT_DIR}/coverage.out"

	@echo "Running gosec" | tee -a "${ARTIFACT_DIR}/sonar.log"
	gosec -fmt sonarqube -out gosec.json -no-fail ./...
	@cp gosec.json "${ARTIFACT_DIR}/sonar_gosec.json"

	@if [[ "${REPO_OWNER}" != "openshift" ]] && [[ "${REPO_NAME}" != "release" ]]; then \
		echo "Fetch ${REPO_OWNER}/${REPO_NAME} and rebase the commits onto ${PULL_BASE_REF}" | tee -a "${ARTIFACT_DIR}/sonar.log"; \
		git remote add origin https://github.com/${REPO_OWNER}/${REPO_NAME} | tee -a "${ARTIFACT_DIR}/sonar.log"; \
		git fetch origin | tee -a "${ARTIFACT_DIR}/sonar.log"; \
		git rebase origin/${PULL_BASE_REF} | tee -a "${ARTIFACT_DIR}/sonar.log"; \
		if [[ "${JOB_TYPE}" == "presubmit" ]]; then \
			BRANCH=$(shell curl -s "https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/pulls/${PULL_NUMBER}" | jq ".head.ref"); \
			echo "Creating and checking out to branch $${BRANCH}" | tee -a "${ARTIFACT_DIR}/sonar.log"; \
			git branch $${BRANCH} | tee -a "${ARTIFACT_DIR}/sonar.log"; \
			git reset --hard origin/${PULL_BASE_REF} | tee -a "${ARTIFACT_DIR}/sonar.log"; \
			git checkout $${BRANCH} | tee -a "${ARTIFACT_DIR}/sonar.log"; \
		fi; \
	fi

	@echo "Starting Sonar Scanner" | tee -a "${ARTIFACT_DIR}/sonar.log"
	@echo "SONAR_ORG=${SONAR_ORG}" | tee -a "${ARTIFACT_DIR}/sonar.log"
	@echo "JOB_TYPE=${JOB_TYPE}" | tee -a "${ARTIFACT_DIR}/sonar.log"
	@echo "PULL_BASE_REF=${PULL_BASE_REF}" | tee -a "${ARTIFACT_DIR}/sonar.log"
	@if [[ "${JOB_TYPE}" == "presubmit" ]]; then \
		echo "PULL_NUMBER=${PULL_NUMBER}" | tee -a "${ARTIFACT_DIR}/sonar.log" ; \
		echo "REPO_OWNER=${REPO_OWNER}" | tee -a "${ARTIFACT_DIR}/sonar.log" ; \
		echo "REPO_NAME=${REPO_NAME}" | tee -a "${ARTIFACT_DIR}/sonar.log" ; \
		echo "BRANCH=$(shell curl -s "https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/pulls/${PULL_NUMBER}" | jq ".head.ref")" | tee -a "${ARTIFACT_DIR}/sonar.log" ; \
		echo "sonar.pullrequest.base=${PULL_BASE_REF}" >> sonar-project.properties ; \
		echo "sonar.pullrequest.key=${PULL_NUMBER}" >> sonar-project.properties ; \
		echo "sonar.pullrequest.branch=$(shell curl -s "https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/pulls/${PULL_NUMBER}" | jq ".head.ref")" >> sonar-project.properties ; \
	elif [[ "${JOB_TYPE}" == "postsubmit" ]]; then \
		echo "sonar.branch.name=${PULL_BASE_REF}" >> sonar-project.properties ; \
	else \
		echo "Invalid JOB_TYPE ${JOB_TYPE}" | tee -a "${ARTIFACT_DIR}/sonar.log" ; \
		exit 1 ; \
	fi

	@sonar-scanner -Dsonar.login=${SONAR_TOKEN} \
		-Dsonar.organization="${SONAR_ORG}" \
		-Dsonar.host.url=https://sonarcloud.io \
		-Dsonar.qualitygate.wait=true \
		-Dsonar.qualitygate.timeout=450 \
		--debug > >(tee -a "${ARTIFACT_DIR}/sonar.log")


# This expects that your code uses Jest to execute tests.
# Add this field to your jest.config.js file to generate the report:
#     testResultProcessor: 'jest-sonar-reporter',
# It must be run before make component/test/unit.
.PHONY: sonar/js/jest-init
## Install npm module to make Sonar test reports in Jest on Travis. This will not be run during local development.
sonar/js/jest-init:
	npm install -D jest-sonar-reporter


# Test reports and code coverage must be generated before running the scanner.
# It must be run after make component/test/unit.
.PHONY: sonar/js
## Runs the SonarCloud analysis for JavaScript on Travis. This will not be run during local development.
sonar/js:
	unset SONARQUBE_SCANNER_PARAMS
	sonar-scanner --debug


.PHONY: sonar/js/prow
## Run SonarCloud analysis for Javascript on Prow CI. This will not be run during local development.
sonar/js/prow: SHELL:=/bin/bash
sonar/js/prow: export SONAR_TOKEN="$(shell cat /etc/sonarcloud/token 2> /dev/null)"
sonar/js/prow:
	@echo "Verify sonar token from kube secret has been mounted" | tee -a "${ARTIFACT_DIR}/sonar.log"
	@if [[ -z "${SONAR_TOKEN}" ]] ; then echo "---> ERROR: SONAR_TOKEN is not set" | tee -a "${ARTIFACT_DIR}/sonar.log" ; exit 1 ; fi

	@if [[ "${REPO_OWNER}" != "openshift" ]] && [[ "${REPO_NAME}" != "release" ]]; then \
		echo "Fetch ${REPO_OWNER}/${REPO_NAME} and rebase the commits onto ${PULL_BASE_REF}" | tee -a "${ARTIFACT_DIR}/sonar.log"; \
		git remote add origin https://github.com/${REPO_OWNER}/${REPO_NAME} | tee -a "${ARTIFACT_DIR}/sonar.log"; \
		git fetch origin | tee -a "${ARTIFACT_DIR}/sonar.log"; \
		git rebase origin/${PULL_BASE_REF} | tee -a "${ARTIFACT_DIR}/sonar.log"; \
		if [[ "${JOB_TYPE}" == "presubmit" ]]; then \
			BRANCH=$(shell curl -s "https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/pulls/${PULL_NUMBER}" | jq ".head.ref"); \
			echo "Creating and checking out to branch $${BRANCH}" | tee -a "${ARTIFACT_DIR}/sonar.log"; \
			git branch $${BRANCH} | tee -a "${ARTIFACT_DIR}/sonar.log"; \
			git reset --hard origin/${PULL_BASE_REF} | tee -a "${ARTIFACT_DIR}/sonar.log"; \
			git checkout $${BRANCH} | tee -a "${ARTIFACT_DIR}/sonar.log"; \
		fi; \
	fi

	@echo "Starting Sonar Scanner" | tee -a "${ARTIFACT_DIR}/sonar.log"
	@echo "SONAR_ORG=${SONAR_ORG}" | tee -a "${ARTIFACT_DIR}/sonar.log"
	@echo "JOB_TYPE=${JOB_TYPE}" | tee -a "${ARTIFACT_DIR}/sonar.log"
	@echo "PULL_BASE_REF=${PULL_BASE_REF}" | tee -a "${ARTIFACT_DIR}/sonar.log"
	@if [[ "${JOB_TYPE}" == "presubmit" ]]; then \
		echo "PULL_NUMBER=${PULL_NUMBER}" | tee -a "${ARTIFACT_DIR}/sonar.log" ; \
		echo "REPO_OWNER=${REPO_OWNER}" | tee -a "${ARTIFACT_DIR}/sonar.log" ; \
		echo "REPO_NAME=${REPO_NAME}" | tee -a "${ARTIFACT_DIR}/sonar.log" ; \
		echo "BRANCH=$(shell curl -s "https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/pulls/${PULL_NUMBER}" | jq ".head.ref")" | tee -a "${ARTIFACT_DIR}/sonar.log" ; \
		echo "sonar.pullrequest.base=${PULL_BASE_REF}" >> sonar-project.properties ; \
		echo "sonar.pullrequest.key=${PULL_NUMBER}" >> sonar-project.properties ; \
		echo "sonar.pullrequest.branch=$(shell curl -s "https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/pulls/${PULL_NUMBER}" | jq ".head.ref")" >> sonar-project.properties ; \
	elif [[ "${JOB_TYPE}" == "postsubmit" ]]; then \
		echo "sonar.branch.name=${PULL_BASE_REF}" >> sonar-project.properties ; \
	else \
		echo "Invalid JOB_TYPE ${JOB_TYPE}" | tee -a "${ARTIFACT_DIR}/sonar.log" ; \
		exit 1 ; \
	fi

	@sonar-scanner -Dsonar.login=${SONAR_TOKEN} \
		-Dsonar.organization="${SONAR_ORG}" \
		-Dsonar.host.url=https://sonarcloud.io \
		-Dsonar.qualitygate.wait=true \
		-Dsonar.qualitygate.timeout=450 \
		--debug > >(tee -a "${ARTIFACT_DIR}/sonar.log")


.PHONY: sonar/py/prow
## Run SonarCloud analysis for Javascript on Prow CI. This will not be run during local development.
sonar/py/prow: SHELL:=/bin/bash
sonar/py/prow: export SONAR_TOKEN="$(shell cat /etc/sonarcloud/token 2> /dev/null)"
sonar/py/prow:
	@echo "Verify sonar token from kube secret has been mounted" | tee -a "${ARTIFACT_DIR}/sonar.log"
	@if [[ -z "${SONAR_TOKEN}" ]] ; then echo "---> ERROR: SONAR_TOKEN is not set" | tee -a "${ARTIFACT_DIR}/sonar.log" ; exit 1 ; fi

	@if [[ "${REPO_OWNER}" != "openshift" ]] && [[ "${REPO_NAME}" != "release" ]]; then \
		echo "Fetch ${REPO_OWNER}/${REPO_NAME} and rebase the commits onto ${PULL_BASE_REF}" | tee -a "${ARTIFACT_DIR}/sonar.log"; \
		git remote add origin https://github.com/${REPO_OWNER}/${REPO_NAME} | tee -a "${ARTIFACT_DIR}/sonar.log"; \
		git fetch origin | tee -a "${ARTIFACT_DIR}/sonar.log"; \
		git rebase origin/${PULL_BASE_REF} | tee -a "${ARTIFACT_DIR}/sonar.log"; \
		if [[ "${JOB_TYPE}" == "presubmit" ]]; then \
			BRANCH=$(shell curl -s "https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/pulls/${PULL_NUMBER}" | jq ".head.ref"); \
			echo "Creating and checking out to branch $${BRANCH}" | tee -a "${ARTIFACT_DIR}/sonar.log"; \
			git branch $${BRANCH} | tee -a "${ARTIFACT_DIR}/sonar.log"; \
			git reset --hard origin/${PULL_BASE_REF} | tee -a "${ARTIFACT_DIR}/sonar.log"; \
			git checkout $${BRANCH} | tee -a "${ARTIFACT_DIR}/sonar.log"; \
		fi; \
	fi

	@echo "Starting Sonar Scanner" | tee -a "${ARTIFACT_DIR}/sonar.log"
	@echo "SONAR_ORG=${SONAR_ORG}" | tee -a "${ARTIFACT_DIR}/sonar.log"
	@echo "JOB_TYPE=${JOB_TYPE}" | tee -a "${ARTIFACT_DIR}/sonar.log"
	@echo "PULL_BASE_REF=${PULL_BASE_REF}" | tee -a "${ARTIFACT_DIR}/sonar.log"
	@if [[ "${JOB_TYPE}" == "presubmit" ]]; then \
		echo "PULL_NUMBER=${PULL_NUMBER}" | tee -a "${ARTIFACT_DIR}/sonar.log" ; \
		echo "REPO_OWNER=${REPO_OWNER}" | tee -a "${ARTIFACT_DIR}/sonar.log" ; \
		echo "REPO_NAME=${REPO_NAME}" | tee -a "${ARTIFACT_DIR}/sonar.log" ; \
		echo "BRANCH=$(shell curl -s "https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/pulls/${PULL_NUMBER}" | jq ".head.ref")" | tee -a "${ARTIFACT_DIR}/sonar.log" ; \
		echo "sonar.pullrequest.base=${PULL_BASE_REF}" >> sonar-project.properties ; \
		echo "sonar.pullrequest.key=${PULL_NUMBER}" >> sonar-project.properties ; \
		echo "sonar.pullrequest.branch=$(shell curl -s "https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/pulls/${PULL_NUMBER}" | jq ".head.ref")" >> sonar-project.properties ; \
	elif [[ "${JOB_TYPE}" == "postsubmit" ]]; then \
		echo "sonar.branch.name=${PULL_BASE_REF}" >> sonar-project.properties ; \
	else \
		echo "Invalid JOB_TYPE ${JOB_TYPE}" | tee -a "${ARTIFACT_DIR}/sonar.log" ; \
		exit 1 ; \
	fi

	@sonar-scanner -Dsonar.login=${SONAR_TOKEN} \
		-Dsonar.organization="${SONAR_ORG}" \
		-Dsonar.host.url=https://sonarcloud.io \
		-Dsonar.qualitygate.wait=true \
		-Dsonar.qualitygate.timeout=450 \
		--debug > >(tee -a "${ARTIFACT_DIR}/sonar.log")
