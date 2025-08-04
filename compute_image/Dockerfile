FROM ghcr.io/warewulf/warewulf-rockylinux:9.4

RUN dnf -y install http://repos.openhpc.community/OpenHPC/3/EL_9/x86_64/ohpc-release-3-1.el9.x86_64.rpm \
    && dnf -y update \
    && dnf install -y --allowerasing \
    ohpc-base-compute \
    ohpc-slurm-client \
    chrony \
    lmod-ohpc \
    apptainer

RUN systemctl enable munge
RUN systemctl enable slurmd

# Just pulling minimal container for testing
RUN apptainer pull docker://ghcr.io/containerd/alpine:latest