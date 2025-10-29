FROM image-registry.openshift-image-registry.svc:5000/openshift/driver-toolkit:latest

RUN mkdir /source
WORKDIR /source
COPY entitlement.pem /etc/pki/entitlement/entitlement.pem
COPY entitlement-key.pem /etc/pki/entitlement/entitlement-key.pem
RUN chmod 600 /etc/pki/entitlement/entitlement-key.pem
COPY lustre-*.tar.gz lustre-client.tar.gz

RUN dnf install -y gcc flex bison kernel-devel python3 python3-devel python3-setuptools \
elfutils-libelf-devel keyutils-libs keyutils-libs-devel libmount-devel libyaml-devel json-c-devel \
rpm-build kernel-rpm-macros kernel-abi-stablelists libselinux-devel libtool \
--enablerepo codeready-builder-for-rhel-9-x86_64-rpms --enablerepo codeready-builder-for-rhel-9-x86_64-eus-rpms
