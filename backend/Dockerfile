# syntax=docker/dockerfile:1

# ---- build: install deps into a venv (no pip cache kept) ----
FROM python:3.10-slim AS build
ENV PIP_NO_CACHE_DIR=1 PIP_DISABLE_PIP_VERSION_CHECK=1
RUN python -m venv /venv
ENV PATH="/venv/bin:$PATH"
COPY requirements.docker.txt .
RUN pip install -r requirements.docker.txt

# ---- runtime ----
FROM python:3.10-slim
ENV PATH="/venv/bin:$PATH" \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    TF_CPP_MIN_LOG_LEVEL=2 \
    HOME=/tmp \
    DJANGO_DEBUG=0
WORKDIR /app
RUN useradd -r -u 10001 app && mkdir -p /data/db /data/media && chown -R app /data
COPY --from=build /venv /venv
COPY --chown=app . .
RUN chmod +x docker-entrypoint.sh && python manage.py collectstatic --noinput
USER app
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=5s --start-period=60s \
  CMD python -c "import socket; socket.create_connection(('127.0.0.1', 8000), 3)" || exit 1
ENTRYPOINT ["./docker-entrypoint.sh"]
# 1 worker + threads: each worker loads the TF model into RAM, so keep workers low
CMD ["gunicorn", "agri_sahayak.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "1", "--threads", "4", "--timeout", "120"]
