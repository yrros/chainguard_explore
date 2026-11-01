# syntax=docker/dockerfile:1

# Build stage: the -dev variant has a shell, pip, apk and a compiler.
FROM cgr.dev/chainguard/python:latest-dev AS build

USER root

# pg_config and libpq headers to compile psycopg2. Chainguard Libraries has no
# prebuilt wheels for this Python version yet, so C extensions build from source.
RUN apk add --no-cache postgresql-18-dev

RUN python -m venv /venv
ENV PATH="/venv/bin:$PATH"

WORKDIR /app
COPY requirements.txt .

# Chainguard Libraries credentials are mounted as a build secret for this step
# only, so they never land in an image layer or the build history.
RUN --mount=type=secret,id=netrc,target=/root/.netrc,required=true \
    pip install --no-cache-dir \
      --index-url https://libraries.cgr.dev/python/simple/ \
      -r requirements.txt

# Runtime stage: no shell or package manager, runs as nonroot.
FROM cgr.dev/chainguard/python:latest

# psycopg2 links against libpq at runtime. These are libpq and its shared library
# dependencies that the runtime image doesn't already have (from `ld.so --list`).
COPY --from=build \
    /usr/lib/libpq.so.5* \
    /usr/lib/libgssapi_krb5.so.2* \
    /usr/lib/libkrb5.so.3* \
    /usr/lib/libk5crypto.so.3* \
    /usr/lib/libkrb5support.so.0* \
    /usr/lib/libcom_err.so.2* \
    /usr/lib/libkeyutils.so.1* \
    /usr/lib/libldap.so.2* \
    /usr/lib/liblber.so.2* \
    /usr/lib/libsasl2.so.3* \
    /usr/lib/
COPY --from=build /venv /venv

WORKDIR /app
COPY . .

ENV PATH="/venv/bin:$PATH"
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

EXPOSE 8000

ENTRYPOINT ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
