FROM python:3.9

WORKDIR /app

COPY ./requirements.txt /app/requirements.txt

RUN apt-get update \
    && apt-get install gcc -y \
    && apt-get clean

RUN pip install --no-cache-dir --upgrade  -r /code/requirements.txt

COPY . /app/