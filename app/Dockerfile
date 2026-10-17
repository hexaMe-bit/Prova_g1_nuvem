FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

RUN groupadd --system app \
 && useradd --system --gid app --no-create-home app \
 && mkdir -p /app/uploads \
 && chown app:app /app/uploads

USER app

EXPOSE 5000

CMD [ "gunicorn", "app:app", "--bind", "0.0.0.0:5000" ]