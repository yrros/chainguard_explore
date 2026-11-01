# syntax=docker/dockerfile:1
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    vim \
    netcat-traditional \
    && rm -rf /var/lib/apt/lists/*

COPY . /app

# Chainguard Libraries credentials are mounted as a build secret for this step
# only, so they never land in an image layer or the build history.
RUN --mount=type=secret,id=netrc,target=/root/.netrc,required=true \
    pip install --no-cache-dir \
      --index-url https://libraries.cgr.dev/python/simple/ \
      -r requirements.txt

ENV FLASK_APP=app.py
ENV FLASK_ENV=development
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

EXPOSE 8000

ENTRYPOINT ["/app/entrypoint.sh"]
