FROM python:3.11-slim

WORKDIR /app

COPY . .

RUN apt update && apt install -y apache2-utils

CMD ["bash", "generate-votes.sh"] 