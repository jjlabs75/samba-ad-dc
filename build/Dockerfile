# =============================================================================
# Samba 4 Active Directory Domain Controller
# Base : debian:13-slim
# =============================================================================

FROM debian:13-slim

LABEL maintainer="Jj Labs"
LABEL description="Samba 4 AD DC"
LABEL version="4.22"

# ─── Variables ────────────────────────────────────────────────────────────────
ENV DEBIAN_FRONTEND=noninteractive
ENV TZ=Europe/Paris

# ─── 1. Packages installation ─────────────────────────────────────────────────
RUN apt-get update && apt-get -y install --no-install-recommends \
        samba-ad-dc \
        samba-ad-provision \
        tdb-tools \
        krb5-kdc \
        chrony \
    && rm -rf /var/lib/apt/lists/*

RUN rm /etc/samba/smb.conf && \
    rm -rf /var/lib/samba/private/*

# ─── 2. Files copy ────────────────────────────────────────────────────────────
COPY samba-provision /usr/local/bin/samba-provision
RUN chmod +x /usr/local/bin/samba-provision

COPY samba-join /usr/local/bin/samba-join
RUN chmod +x /usr/local/bin/samba-join

# ─── 3. Entrypoint ────────────────────────────────────────────────────────────
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh


# ─── 4. Exposed ports ─────────────────────────────────────────────────────────
# NTP (chrony)
EXPOSE 123/udp
# DNS
EXPOSE 53/tcp
EXPOSE 53/udp
# Kerberos
EXPOSE 88/tcp
EXPOSE 88/udp
# RPC Endpoint Mapper
EXPOSE 135/tcp
# NetBIOS Name Service
EXPOSE 137/udp
# NetBIOS Datagram Service
EXPOSE 138/udp
# NetBIOS Session Service
EXPOSE 139/tcp
# LDAP
EXPOSE 389/tcp
EXPOSE 389/udp
# SMB over TCP (CIFS)
EXPOSE 445/tcp
# Kerberos kpasswd
EXPOSE 464/tcp
EXPOSE 464/udp
# LDAPS
EXPOSE 636/tcp
# Global Catalog
EXPOSE 3268/tcp
# Global Catalog SSL
EXPOSE 3269/tcp
# RPC dynamique
EXPOSE 49152-65535/tcp

# ─── 5. Volumes ───────────────────────────────────────────────────────────────
VOLUME ["/etc/samba", "/var/lib/samba", "/var/log/samba", "/backups"]

# ─── 6. Entrypoint ───────────────────────────────────────────────────────────
ENTRYPOINT ["/entrypoint.sh"]
