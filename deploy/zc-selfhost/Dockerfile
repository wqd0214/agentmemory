ARG III_VERSION=0.22.1

FROM iiidev/iii:${III_VERSION} AS iii-image

FROM node:22-slim

ARG III_VERSION=0.22.1
ARG III_SDK_VERSION=0.22.1

RUN apt-get update \
 && apt-get install -y --no-install-recommends openssl ca-certificates tini gosu curl \
 && rm -rf /var/lib/apt/lists/*

COPY --from=iii-image /app/iii /usr/local/bin/iii
RUN iii --version 2>&1 | grep -qF "${III_VERSION}" \
 || { echo "the iiidev/iii:${III_VERSION} image does not contain iii ${III_VERSION}" >&2; exit 1; }

WORKDIR /opt/agentmemory
# zc-selfhost: install OUR built package (fork with patches) instead of the
# registry release. --legacy-peer-deps works around npm 10.9.x arborist
# "Cannot read properties of null (reading 'edgesOut')".
COPY agentmemory-0.9.30-zc.tgz /tmp/pkg.tgz
RUN [ "${III_VERSION}" = "${III_SDK_VERSION}" ] \
 || { echo "III_VERSION=${III_VERSION} and III_SDK_VERSION=${III_SDK_VERSION} must be the same release: the agentmemory worker speaks the wire protocol of the engine it was built against, so override both together" >&2; exit 1; }
RUN printf '{"name":"agentmemory-deploy","version":"1.0.0","private":true,"overrides":{"iii-sdk":"%s"}}\n' "${III_SDK_VERSION}" > package.json \
 && npm install /tmp/pkg.tgz --omit=optional --no-fund --no-audit --legacy-peer-deps \
 && rm -f /tmp/pkg.tgz \
 && ln -s /opt/agentmemory/node_modules/.bin/agentmemory /usr/local/bin/agentmemory

ENV AGENTMEMORY_III_VERSION=${III_VERSION} \
    TINI_SUBREAPER=1

COPY --chmod=0755 entrypoint.sh /usr/local/bin/agentmemory-entrypoint.sh

EXPOSE 3111

# Large stores need minutes of first-boot index building before /agentmemory/livez
# answers; give the healthcheck a long start period (readiness gate itself is
# tuned at `docker run` via AGENTMEMORY_WORKER_READY_TIMEOUT_MS).
HEALTHCHECK --interval=30s --timeout=5s --start-period=900s --retries=3 \
  CMD curl -fsS http://127.0.0.1:3111/agentmemory/livez || exit 1

ENTRYPOINT ["/usr/bin/tini", "--", "/usr/local/bin/agentmemory-entrypoint.sh"]
