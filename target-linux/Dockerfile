FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

# Instalace SSH, Apache a základních nástrojů pro studenty (nano, net-tools)
RUN apt-get update && apt-get install -y \
    openssh-server \
    apache2 \
    nano \
    net-tools \
    iproute2 \
    && rm -rf /var/lib/apt/lists/*

# Příprava SSH a nastavení slabého hesla pro root
RUN mkdir /var/run/sshd && \
    echo 'root:student123' | chpasswd && \
    sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config

# Úmyslné oslabení SSH konfigurace (zastaralé šifry / MACs pro detekci skenerem)
RUN echo "Ciphers aes128-cbc,3des-cbc,aes128-ctr" >> /etc/ssh/sshd_config && \
    echo "MACs hmac-sha1,hmac-sha2-256" >> /etc/ssh/sshd_config

EXPOSE 22 80

# Spuštění Apache i SSH najednou v popředí
CMD service apache2 start && /usr/sbin/sshd -D