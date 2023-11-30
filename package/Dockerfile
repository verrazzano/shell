FROM ghcr.io/oracle/oraclelinux:8-slim as helm
ARG HELM_VERSION=3.9.0-rancher1
RUN microdnf update -y --setopt=install_weak_deps=0 --setopt=tsflags=nodocs && \
    microdnf install -y make git && \
    microdnf install -y golang-1.20.10-1.module+el8.9.0+90068+08775a1f.x86_64 && \
    # Rancher uses its own Helm fork - build and install our BFS of that fork.
    git -C / clone --branch oracle/release/${HELM_VERSION} --depth=1 https://github.com/verrazzano/helm && \
    make -C /helm && \
    microdnf clean all && \
    rm -rf /var/cache/yum /var/lib/rpm/*

FROM ghcr.io/oracle/oraclelinux:8 as build
ARG ARCH=amd64
RUN dnf update -y --setopt=install_weak_deps=0 --setopt=tsflags=nodocs && \
    dnf install -y gzip && \
    dnf clean all && \
    rm -rf /var/cache/yum /var/lib/rpm/*

ENV KUSTOMIZE_VERSION v5.0.2
ENV KUSTOMIZE_URL=https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize/${KUSTOMIZE_VERSION}/kustomize_${KUSTOMIZE_VERSION}_linux_${ARCH}.tar.gz
ENV K9S_VERSION v0.26.7
RUN if [ "${ARCH}" = "amd64" ]; then ARCH=x86_64; fi && \
    curl -sfL https://github.com/derailed/k9s/releases/download/${K9S_VERSION}/k9s_Linux_${ARCH}.tar.gz | tar xvzf -
RUN if [ "${ARCH}" = "amd64" ] || [ "${ARCH}" = "arm64" ] || [ "${ARCH}" = "s390x" ]; then \
    curl -sLf ${KUSTOMIZE_URL} | tar -xzf - && chmod +x kustomize; \
    fi

FROM ghcr.io/oracle/oraclelinux:8-slim AS install-deps

RUN microdnf update -y --setopt=install_weak_deps=0 --setopt=tsflags=nodocs && \
    microdnf install -y bash-completion gzip jq unzip vim wget && \
    microdnf install -y dnf && \
    microdnf install -y 'dnf-command(config-manager)' && \
    dnf config-manager --add-repo http://yum.oracle.com/repo/OracleLinux/OL8/olcne17/x86_64 && \
    microdnf install -y kubectl-1.26.6-1.el8.x86_64 && \
    microdnf clean all && \
    rm -rf /var/cache/yum /var/lib/rpm/* && \
    ln -s /usr/bin/vim /usr/bin/vi && \
    rm -rf /tmp/* /var/tmp/* /usr/share/doc/packages/* /usr/share/doc/manual/* /var/log/*

RUN echo 'shell:x:1000:1000:shell,,,:/home/shell:/bin/bash' > /etc/passwd && \
    echo 'shell:x:1000:' > /etc/group && \
    mkdir /home/shell && \
    echo '. /etc/profile.d/bash_completion.sh' >> /home/shell/.bashrc && \
    echo 'alias k="kubectl"' >> /home/shell/.bashrc && \
    echo 'alias ks="kubectl -n kube-system"' >> /home/shell/.bashrc && \
    echo 'source <(kubectl completion bash)' >> /home/shell/.bashrc && \
    echo 'complete -o default -F __start_kubectl k' >> /home/shell/.bashrc && \
    echo 'PS1="> "' >> /home/shell/.bashrc && \
    mkdir /home/shell/.kube && \
    chown -R shell /home/shell && \
    chmod 700 /run && \
    mkdir -p /LICENSES

FROM ghcr.io/verrazzano/verrazzano-base:v1.0.0-20231025034637-742ef97

COPY --from=install-deps /etc/passwd /etc/passwd
COPY --from=install-deps /etc/group /etc/group

COPY --from=helm ./helm/bin/helm /usr/local/bin/
COPY --from=build /k9s ./kustomize* /usr/local/bin/
COPY --from=build /bin/xargs /bin

# Copy bash completion files
COPY --from=install-deps /etc/profile.d/bash_completion.sh /etc/profile.d/bash_completion.sh
COPY --from=install-deps /usr/share/bash-completion/bash_completion /usr/share/bash-completion/bash_completion

# Copy binaries
COPY --from=install-deps /usr/bin/vim /usr/bin/vim
COPY --from=install-deps /usr/bin/vi /usr/bin/vi
COPY --from=install-deps /usr/bin/gzip /usr/bin/gzip
COPY --from=install-deps /usr/bin/jq /usr/bin/jq
COPY --from=install-deps /usr/bin/unzip /usr/bin/unzip
COPY --from=install-deps /usr/bin/wget /usr/bin/wget

COPY --from=install-deps /home/shell /home/shell

COPY package/helm-cmd package/welcome /usr/local/bin/
COPY kustomize.sh /home/shell/
COPY THIRD_PARTY_LICENSES.txt LICENSE SECURITY.md /LICENSES/
USER 1000
WORKDIR /home/shell
CMD ["welcome"]
