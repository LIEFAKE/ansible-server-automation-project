# A small "server": Ubuntu 24.04 with systemd and SSH, so Ansible can treat it like a real VM.
# Infrastructure helper: lets the playbooks run against "real" servers locally.
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      systemd systemd-sysv openssh-server python3 python3-apt sudo ca-certificates \
 && apt-get clean \
 && rm -f /etc/systemd/system/getty.target.wants/* \
 && systemctl enable ssh \
 && systemctl mask systemd-binfmt.service proc-sys-fs-binfmt_misc.automount proc-sys-fs-binfmt_misc.mount

# Why the mask above: the container is privileged and shares the WSL kernel.
# Its systemd-binfmt would wipe the host's binfmt rules on every start,
# including WSLInterop, and then `code .` / any .exe stops working in WSL.

# The account Ansible logs in as. Password "ansible" is only for the first
# (bootstrap) connection; site.yml turns password login off.
# Passwordless sudo is normal for an automation account.
RUN useradd -m -s /bin/bash ansible \
 && echo 'ansible:ansible' | chpasswd \
 && echo 'ansible ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/ansible \
 && chmod 0440 /etc/sudoers.d/ansible

STOPSIGNAL SIGRTMIN+3
CMD ["/lib/systemd/systemd"]
