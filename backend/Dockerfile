FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
COPY crypto_tools.py .
# COPY .env .          ← ELIMINAR esta línea (la API key ahora viene del K8s Secret)

EXPOSE 80
CMD ["python", "app.py"]
