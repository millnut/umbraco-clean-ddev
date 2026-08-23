#ddev-silent-no-warn
# Additive to DDEV's web image, which is prepended as `FROM $BASE_IMAGE`.
# Do not add a FROM here. It starts a fresh stage and discards ddev-webserver's
# user setup, supervisord, healthcheck and tooling, all of which then have to be
# hand-rolled back.

# Debian trixie carries no .NET packages at all, so the SDK comes from Microsoft's
# feed. Register it with the .deb: the microsoft.asc and `gpg --dearmor` route
# fails on trixie with NO_PUBKEY EE4D7792F748182B, because that key no longer
# signs this repo. Only .NET 10 publishes arm64 there, so Apple Silicon cannot
# drop below 10.0. ACCEPT_EULA answers the prompts from mssql-tools18 and the
# msodbcsql18 it pulls in.
RUN curl -fsSL https://packages.microsoft.com/config/debian/13/packages-microsoft-prod.deb -o /tmp/pmp.deb \
    && dpkg -i /tmp/pmp.deb \
    && rm /tmp/pmp.deb \
    && apt-get update \
    && ACCEPT_EULA=Y apt-get install -y dotnet-sdk-10.0 mssql-tools18 \
    && rm -rf /var/lib/apt/lists/*

# No apt package exists for this, and the bacpac-* commands depend on it
RUN dotnet tool install --tool-path /usr/local/share/dotnet-tools microsoft.sqlpackage

ENV PATH="$PATH:/usr/local/share/dotnet-tools:/opt/mssql-tools18/bin"
