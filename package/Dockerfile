FROM ghcr.io/oracle/oraclelinux:7-slim as helm
ARG HELM_VERSION=3.9.0-rancher1
RUN yum install -y make && \
    yum install -y oracle-golang-release-el7 && \
    yum-config-manager --enable ol7_developer_golang119 && \
    yum install -y golang-1.19-1.0.1.el7.x86_64 && \
    # software collections repo needed for git 2.x on OL7
    yum-config-manager --add-repo=http://yum.oracle.com/repo/OracleLinux/OL7/SoftwareCollections/x86_64 && \
    yum install -y rh-git227 && \
    # Set up needed to ensure git 2.27 from rh-git227 is on the path (gitjob original v0.1.30 image uses git version 2.35.3)
    ln /opt/rh/rh-git227/enable /etc/profile.d/git.sh && \
    source /etc/profile.d/git.sh && \
    git version && \
    # Rancher uses its own Helm fork - build and install our BFS of that fork.
    git -C / clone --branch oracle/release/${HELM_VERSION} --depth=1 https://github.com/verrazzano/helm && \
    make -C /helm

FROM ghcr.io/oracle/oraclelinux:7-slim as build
ARG ARCH=amd64
RUN yum-config-manager --enable ol7_optional_latest && \
    yum-config-manager --enable ol7_addons && \
    yum update -y && \
    yum install -y bash curl gzip tar

ENV KUSTOMIZE_VERSION v5.0.2
ENV KUSTOMIZE_URL=https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize/${KUSTOMIZE_VERSION}/kustomize_${KUSTOMIZE_VERSION}_linux_${ARCH}.tar.gz
ENV K9S_VERSION v0.26.7
RUN if [ "${ARCH}" = "amd64" ]; then ARCH=x86_64; fi && \
    curl -sfL https://github.com/derailed/k9s/releases/download/${K9S_VERSION}/k9s_Linux_${ARCH}.tar.gz | tar xvzf -
RUN if [ "${ARCH}" = "amd64" ] || [ "${ARCH}" = "arm64" ] || [ "${ARCH}" = "s390x" ]; then \
    curl -sLf ${KUSTOMIZE_URL} | tar -xzf - && chmod +x kustomize; \
    fi

FROM ghcr.io/oracle/oraclelinux:7-slim

RUN yum-config-manager --enable ol7_optional_latest && \
    yum-config-manager --enable ol7_addons && \
    yum update -y && \
    # NOTE - psmisc is needed for "killall" command, which Rancher uses to kill the proxy container on helm operation pods
    yum install -y bash bash-completion gzip jq tar unzip vim wget psmisc && \
    yum-config-manager --add-repo https://yum.oracle.com/repo/OracleLinux/OL7/olcne16/x86_64 && \
    yum -y install kubectl-1.25.11-1.el7.x86_64 && \
    yum clean all && \
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
COPY package/helm-cmd package/welcome /usr/local/bin/
COPY kustomize.sh /home/shell/
COPY THIRD_PARTY_LICENSES.txt LICENSE SECURITY.md /LICENSES/
USER 1000
WORKDIR /home/shell
CMD ["welcome"]
