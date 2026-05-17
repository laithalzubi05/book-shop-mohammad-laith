FROM python:3.11-slim

WORKDIR /app

COPY artifact.tar.gz /tmp/artifact.tar.gz
RUN tar -xzf /tmp/artifact.tar.gz -C /app && rm /tmp/artifact.tar.gz

RUN pip install --no-cache-dir -r requirements.txt

EXPOSE 8000

CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]