# FestPass backend - Stage 1 build on the developers' Python 3.9 base.
FROM python:3.9

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /app

# Install dependencies first so this layer is cached until requirements.txt changes.
COPY requirements.txt .
RUN pip install -r requirements.txt

# Application code changes most often, so it is copied last.
COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
