FROM ghcr.io/verrazzano/golang:v1.20.12 as helm
ARG HELM_VERSION=mgianata/vz-11511
RUN dnf config-manager --enable ol8_appstream && \
    dnf install -y make git && \
    # Rancher uses its own Helm fork - build and install our BFS of that fork.
    git -C / clone --branch ${HELM_VERSION} --depth=1 https://github.com/verrazzano/helm && \
    make -C /helm

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

FROM ghcr.io/oracle/oraclelinux:8-slim

RUN microdnf update -y --setopt=install_weak_deps=0 --setopt=tsflags=nodocs && \
    microdnf install -y bash-completion gzip jq unzip vim wget && \
    microdnf install -y dnf && \
    microdnf install -y 'dnf-command(config-manager)' && \
    dnf config-manager --add-repo http://yum.oracle.com/repo/OracleLinux/OL8/olcne17/x86_64 && \
    microdnf install -y kubectl-1.26.10-2.el8.x86_64 && \
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

COPY --from=helm ./helm/bin/helm /usr/local/bin/
COPY --from=build /k9s ./kustomize* /usr/local/bin/
COPY --from=build /bin/xargs /bin
COPY package/helm-cmd package/welcome /usr/local/bin/
COPY kustomize.sh /home/shell/
COPY THIRD_PARTY_LICENSES.txt LICENSE SECURITY.md /LICENSES/
USER 1000
WORKDIR /home/shell
CMD ["welcome"]
