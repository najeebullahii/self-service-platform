FROM python:3.14-slim

LABEL org.opencontainers.image.source="https://github.com/${{ values.owner }}/${{ values.name }}"

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Patch operating system packages at build time
RUN apt-get update && apt-get upgrade -y && rm -rf /var/lib/apt/lists/*

# The app needs no package manager at run time, so remove it from the final image
RUN pip uninstall -y pip setuptools wheel

# Run as a restricted user, not as root
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app
COPY app.py .
RUN chown -R appuser:appgroup /app
USER appuser

EXPOSE 8080
CMD ["python", "app.py"]
