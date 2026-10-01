# builder-base holds kernel-independent inputs for KMM in-cluster builds:
# Lustre tarball and RHSM entitlement certs.
#
# Do not install gcc/kernel-devel here — that pins the image to the kernel
# of this DTK. KMM Dockerfiles COPY these files, then compile against
# ${DTK_AUTO} for the node's kernel. Rebuild only when the tarball or
# entitlements change.
FROM image-registry.openshift-image-registry.svc:5000/openshift/driver-toolkit:latest

RUN mkdir /source
WORKDIR /source
COPY entitlement.pem /etc/pki/entitlement/entitlement.pem
COPY entitlement-key.pem /etc/pki/entitlement/entitlement-key.pem
RUN chmod 600 /etc/pki/entitlement/entitlement-key.pem
COPY lustre-*.tar.gz lustre-client.tar.gz
