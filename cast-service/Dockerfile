FROM python:3.8-slim

WORKDIR /app

COPY ./requirements.txt /app/requirements.txt

RUN apt-get update \
    && apt-get install gcc -y \
    && apt-get clean

RUN pip install -r /app/requirements.txt \
    && rm -rf /root/.cache/pip

COPY . /app/

# Use asyncio to avoid the "no current event loop" startup error caused by the legacy Uvicorn/uvloop combination in this project.
CMD ["uvicorn", "app.main:app", "--loop", "asyncio", "--host", "0.0.0.0", "--port", "8000"]
