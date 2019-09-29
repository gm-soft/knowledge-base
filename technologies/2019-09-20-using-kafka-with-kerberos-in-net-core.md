---
layout: post
title: Using Kafka with Kerberos authentication from .NET Core
category: technologies
tags: [copypaste, technologies, development, kafka, .net, .net core]
---

Source: [.NET or not .NET](https://shatl.wordpress.com/2018/10/30/using-kafka-with-kerberos-authentication-from-net-core/)

---

# Issues

- nuget version of librdkafka (as of 0.11.5) does not support Kerberos authentication out of box, so custom library needs to be build and injected into deployed binaries.
- Running on Linux requires **keytab** file, while it does not supported on Windows, where current principal is used to identify client.

# Building LibrdKafka with Kerberos support

This can be done in docker

### DOCKERFILE

```Dockerfile
    FROM microsoft/dotnet:2.1-aspnetcore-runtime AS base
    WORKDIR /app
    
    FROM microsoft/dotnet:2.1-sdk as build
    # Download dependency for building librdkafka
    RUN apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF \
    && echo "deb http://download.mono-project.com/repo/debian stretch main" | tee /etc/apt/sources.list.d/mono-official.list \
    && apt-get update && apt-get install -y mono-devel default-jre build-essential libssl-dev libsasl2-2 libsasl2-dev libsasl2-modules-gssapi-mit wget unzip
    
    # Build librdkafkaENV LIBRDKAFKA_VER=0.11.5
    RUN curl -k -L -s https://github.com/edenhill/librdkafka/archive/v${LIBRDKAFKA_VER}.zip -o ./librdkafka.zip
    
    RUN ls -l && cd / && unzip librdkafka.zip && \
    cd librdkafka-${LIBRDKAFKA_VER} && \
    ./configure && \
    make && \
    make install
    
    # Build .net app
    WORKDIR /src
    COPY ["Kafka.Kerberos/Kafka.Kerberos.csproj", "Kafka.Kerberos/"]
    RUN dotnet restore "Kafka.Kerberos/Kafka.Kerberos.csproj"
    COPY . .
    WORKDIR "/src/Kafka.Kerberos"
    RUN dotnet build "Kafka.Kerberos.csproj" -c Release -o /app
    
    FROM build AS publish
    RUN dotnet publish "Kafka.Kerberos.csproj" -c Release -o /app
    
    FROM base AS final
    ENV ASPNETCORE_URLS http://*:5000
    
    # Install runtime dependencies for kerberos
    RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get -y install krb5-user kstart \
    libsasl2-2 libsasl2-modules-gssapi-mit libsasl2-modules \
    && apt-get autoremove
    
    WORKDIR /app
    COPY --from=publish /app .
    COPY ./service.keytab /app/
    RUN rm -f /app/runtimes/linux-x64/native/librdkafka.so
    COPY --from=build /usr/local/lib/librdkafka*.so* /app/runtimes/linux-x64/native/
    
    ENTRYPOINT ["dotnet", "Kafka.Kerberos.dll"]
```


### Producer configuration (LINUX)

To run from Linux, keytab file path and principal must be specified.

**Note:** Principal in settings must match principal in keytab file.

```csharp
    var internalConfig = new Dictionary() {
    	["bootstrap.servers"] = brokerList,
    	["client.id"] = "producer",
    	["security.protocol"] = "SASL_SSL",
    	["api.version.request"] = true,
    	["sasl.kerberos.service.name"] = "kafka",
    	["sasl.kerberos.keytab"] = "/secrets/kafka-producer.keytab",
    	["sasl.kerberos.principal"] = "kafka-producer@DOMAIN",
    };
```

### Producer configuration (Windows)

Windows configuration is a

- No keytab is required;
- No principal is required, current process principal is used.

```csharp
    var internalConfig = new Dictionary() {
    	["bootstrap.servers"] = brokerList,
    	["client.id"] = "producer",
    	["security.protocol"] = "SASL_SSL",
    	["api.version.request"] = true,
    	["sasl.kerberos.service.name"] = "kafka",
    };
```

### Using SSL and self-signed certificate

In case you need to use self-signed certificate, CA path must be provided:

```csharp
    ["ssl.ca.location"] = "/CA/ca-path.pem";
```