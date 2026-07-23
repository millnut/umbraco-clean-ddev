#ddev-silent-no-warn
# Replaces the default PHP/nginx DDEV web container with .NET + Kestrel

FROM mcr.microsoft.com/dotnet/sdk:10.0

# Re-declare so they survive the FROM boundary into this build stage
ARG username
ARG uid
ARG gid

# DDEV's injected scripts shell out to these
RUN apt-get update && apt-get install -y \
    curl \
    ca-certificates \
    git \
    wget \
    vim \
    netcat-openbsd \
    procps \
    sudo \
    && rm -rf /var/lib/apt/lists/*

# sqlpackage enables bacpac import/export against the SQL Server sidecar
RUN dotnet tool install --tool-path /usr/local/share/dotnet-tools microsoft.sqlpackage

# sqlcmd for ad-hoc SQL (e.g., dropping a database before bacpac import)
RUN apt-get update \
    && apt-get install -y gnupg \
    && curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor -o /usr/share/keyrings/microsoft-prod.gpg \
    && curl -fsSL https://packages.microsoft.com/config/debian/12/prod.list | tee /etc/apt/sources.list.d/mssql-release.list \
    && apt-get update \
    && ACCEPT_EULA=Y apt-get install -y mssql-tools18 \
    && rm -rf /var/lib/apt/lists/*

ENV PATH="$PATH:/usr/local/share/dotnet-tools:/opt/mssql-tools18/bin"

# DDEV/Mutagen sync requires this user/group to own mounted files
RUN (groupadd --gid "$gid" "$username" || groupadd "$username" || true) && \
    (useradd -l -m -s "/bin/bash" --gid "$gid" --uid "$uid" "$username" || \
     useradd -l -m -s "/bin/bash" --gid "$gid" "$username" || true) && \
    echo "$username ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# DDEV injects scripts that reference PHP/nginx/Apache paths even in non-PHP containers
RUN mkdir -p /run/php /var/log /etc/php /var/lib/php /tmp/xhprof \
    /usr/local/n /var/run/nginx /var/run/supervisor /var/run/apache2 /var/lock/apache2 \
    /var/log/nginx /var/log/apache2 /var/lib/dpkg/alternatives \
    /home/$username \
    && touch /usr/local/bin/composer \
    && chown $username /home/$username \
    && chmod 777 /run/php /var/log /tmp/xhprof \
    && chmod -R 777 /etc/php /var/lib/php || true

# DDEV's router proxies to 8080; this lets the healthcheck confirm Kestrel is up
RUN printf '#!/bin/bash\ncurl -sf http://localhost:8080/ >/dev/null 2>&1 || exit 1\n' > /healthcheck.sh \
    && chmod +x /healthcheck.sh

WORKDIR /var/www/html
EXPOSE 80 443

# Placeholder so the image is runnable standalone; docker-compose overrides this
CMD ["dotnet", "--version"]
