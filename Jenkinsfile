pipeline {
  agent any

  parameters {
    string(name: 'CI_BUILD_TOOLS_VERSION_OVERRIDE', defaultValue: '', description: 'Override CI Build installer version')
    string(name: 'PROJECT' , defaultValue: 'projects/zint-2dd7cf83', description: 'Name of Project to fuzz')
    string(name: 'TIMEOUT', defaultValue: '300', description: 'Seconds to wait for a finding. If nothing is found within this time span,'+
    'the pipeline run will be considered a success.')
    string(name: 'FINDINGS_TYPE', defaultValue: 'CRASH', description: 'Type of finding to wait for. If found, the pipeline will be marked as failure.')
    string(name: 'FUZZING_SERVER', defaultValue: 'nightly.code-intelligence.com:443', description: 'URL of the CI Fuzz gRPC API.')
    string(name: 'WEB_APP_ADDRESS', defaultValue: 'https://nightly.code-intelligence.com', description: 'URL of the CI Fuzz Web UI.')
    credentials(
      name: 'CI_FUZZ_API_TOKEN',
      credentialType: 'org.jenkinsci.plugins.plaincredentials.impl.StringCredentialsImpl',
      defaultValue: 'CI_FUZZ_API_TOKEN',
      description: 'Access Token for CI Fuzz API.')
  }

  stages {
    stage ('CI Fuzz') {
      steps {
        withCredentials([string(credentialsId: "${CI_FUZZ_API_TOKEN}", variable: 'CI_FUZZ_API_TOKEN')]) {
          sh '''
            CIFUZZ_INSTALL_DIR="$HOME/cifuzz"
            CI_BUILD=${CIFUZZ_INSTALL_DIR}/bin/ci-build
            CICTL=${CIFUZZ_INSTALL_DIR}/bin/cictl
            ### Download CI Fuzz Build Tools
            CIBUILD_VERSION=$(${CI_BUILD} --version | grep -o -P '\\d+\\.\\d+\\.\\d+') || true
            if [ -n "${CI_BUILD_TOOLS_VERSION_OVERRIDE}" ]; then
              CIFUZZ_SERVER_VERSION="${CI_BUILD_TOOLS_VERSION_OVERRIDE}"
            else
              CIFUZZ_SERVER_VERSION= $(curl "$WEB_APP_ADDRESS/v1/version" | grep -o -P '\\d+\\.\\d+\\.\\d+') \
                || ( echo "Problem getting CI Fuzz server version, cannot download/update CI Fuzz build tools!" ; false )
            fi

            if [ "$CIBUILD_VERSION" != "$CIFUZZ_SERVER_VERSION" ]; then
                INSTALLER=ci-fuzz-build-tools-installer-$CIFUZZ_SERVER_VERSION-linux
                curl "https://s3.eu-central-1.amazonaws.com/public.code-intelligence.com/releases/$INSTALLER" -s -o $INSTALLER
                chmod +x $INSTALLER
                rm -rf $CIFUZZ_INSTALL_DIR
                ./$INSTALLER --non-interactive --extract-only --install-dir $CIFUZZ_INSTALL_DIR
            fi

            ### Log in to CI Fuzz
            CICTL_CMD="${CICTL} --server ${FUZZING_SERVER}"
            echo "${CI_FUZZ_API_TOKEN}" | $CICTL_CMD login --quiet

            ### Build Artifact
            ARTIFACT_LOCATION=$HOME/zint-fuzzers.tar.gz
            ${CI_BUILD} fuzzers -o ${ARTIFACT_LOCATION}

            ### Import Artifact
            ARTIFACT_NAME=$(${CICTL_CMD} import artifact ${ARTIFACT_LOCATION} --project-name "${PROJECT}")

            ### Start Campaign Run
            CAMPAIGN_RUN_NAME=$(${CICTL_CMD} start "${ARTIFACT_NAME}")

            ### Observe Campaign Run with timeout, watching for findings
            mkdir -p "${BUILD_TAG}"
            cd "${BUILD_TAG}"

            ${CICTL_CMD} monitor_campaign_run \
              --dashboard_address="${WEB_APP_ADDRESS}" \
              --duration="${TIMEOUT}" \
              --findings_type="${FINDINGS_TYPE}" \
              ${CAMPAIGN_RUN_NAME}
          '''
          }
        }
    }
  }
  post {
    always {
      sh '''
        set -eu

        pwd
        if [ -d "${BUILD_TAG}" ]; then
            cd "${BUILD_TAG}"
        else
            # There isn't a work dir, most likely the fuzzing step failed before creating it
            exit 0
        fi

        # Check if there are any findings
        if ! stat -t finding-*.json > /dev/null 2>&1; then
          # There are no findings, so there's nothing to do
         exit 0
        fi

        JQ="${WORKSPACE}/jq"
        JQ_URL=https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux64
        JQ_CHECKSUM=af986793a515d500ab2d35f8d2aecd656e764504b789b66d7e1a0b727a124c44

        # Check if jq already exists
        if [ -f "${JQ}" ]; then
          # The file already exists. Verify the checksum.
          if ! echo "${JQ_CHECKSUM}" "${JQ}" | sha256sum --check; then
            # The checksum of the existing file doesn't match. Download
            # it again.
            curl -L "${JQ_URL}" -o "${JQ}"
          fi
        else
          # The file doesn't exist yet, download it
          curl -L "${JQ_URL}" -o "${JQ}"
        fi

        # Verify the checksum
        echo "${JQ_CHECKSUM}" "${JQ}" | sha256sum --check

        # Make it executable
        chmod +x "${JQ}"

        # Merge findings into one file
        "${JQ}" --slurp '.' finding-*.json > cifuzz_findings.json
        '''

        archiveArtifacts artifacts: "${BUILD_TAG}/cifuzz_findings.json", fingerprint: true
      }
    }
}
